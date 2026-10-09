# 网页反调试：BOSS 的做法、绕过过程与项目接入示例

BOSS 页面在浏览器调试时反复刷新。查到两套独立检测后，在它们初始化前替换入口，页面恢复正常。下面给出对应代码和自己项目的接入方式。

## 1. BOSS 怎么做的

| 位置 | 检测入口 | 行为 |
| --- | --- | --- |
| SPA 的 webpack 模块 | `noDebug()` | 创建检测器，定时检查 DOM getter、Date/Function 的 `toString` 和控制台输出耗时 |
| 公共脚本 | `Sign.encryptPwd()` | 独立创建另一套检测器；方法名像加密，但实际方法体包含反调试逻辑 |
| 检测命中后的处理 | 检测器回调 | 刷新、关闭窗口、清空或隐藏页面 |

### 检测代码长什么样

下面是从实际逻辑提炼的示例，省略混淆代码及其他检测器，不是原脚本全文：

```js
class DateDetector {
  constructor(onDetected) {
    this.count = 0;
    this.date = new Date();
    this.onDetected = onDetected;
    this.date.toString = () => {
      this.count += 1;
      return '';
    };
  }

  check() {
    this.count = 0;
    console.log(this.date);
    if (this.count >= 2) this.onDetected();
  }
}

const detector = new DateDetector(onDetected);
const timer = setInterval(() => detector.check(), 500);
// 页面退出时：clearInterval(timer)
```

调试工具展示对象时，可能额外调用 `toString()`。检测器利用这个差异计数。Function 检测采用类似方法；DOM 检测则给元素属性设置 getter，观察它是否被读取。

**不是所有浏览器或调试方式都会触发这个示例，也可能误判。** BOSS 使用多个检测器组合，而不是只靠一个 Date 对象。

## 2. 我怎么解决的

### 先找到谁在刷新

通过浏览器调试协议观察 `Page.frameRequestedNavigation`，再读取 Document 请求的 `Network.requestWillBeSent`：

```js
// 已连接、已启用 Page 和 Network 的调试客户端
client.on('Network.requestWillBeSent', event => {
  if (event.type !== 'Document') return;
  if (event.initiator.type !== 'script') return;
  console.log(event.initiator.stack?.callFrames);
});
```

调用栈给出了发起导航的脚本和位置。处理 SPA 的检测后仍然刷新，继续读取调用栈，才找到公共脚本里的第二套检测。

### 在检测初始化前替换入口

扩展的关键配置：

```json
{
  "manifest_version": 3,
  "name": "页面检测调试补丁",
  "version": "1",
  "content_scripts": [{
    "matches": ["https://your-site.example/*"],
    "js": ["guard.js"],
    "run_at": "document_start",
    "world": "MAIN"
  }]
}
```

`document_start` 让补丁提前运行；`MAIN` 让补丁进入页面自身的 JavaScript 环境。上面的域名是占位符，使用时换成自己的测试站点。

实际扩展通过源码特征确认目标，分别处理两个入口：

| 入口 | 拦截位置 | 替换内容 |
| --- | --- | --- |
| webpack 模块 | chunk 队列的 `push`，模块交给 runtime 之前 | 目标模块工厂 |
| 公共脚本 | `window.Sign` 被赋值时 | 已确认的 `encryptPwd` 检测方法 |

webpack 模块替换的核心如下。`modules` 是 chunk 中的模块表，`targetId` 是已经核验的目标编号：

```js
function replaceDetector(modules, targetId, matchesSource) {
  const original = modules[targetId];
  if (typeof original !== 'function') return false;
  if (!matchesSource(Function.prototype.toString.call(original))) {
    return false;
  }

  modules[targetId] = function (module, exports, require) {
    require.r(exports);
    require.d(exports, { noDebug: () => noDebug });
  };
  return true;
}

function noDebug(options, initialize) {
  if (!options?.appName) throw new Error('Unable to obtain information');
  try { initialize?.(); } catch { /* 原入口也捕获初始化异常 */ }
  return { success: true, reason: '', check: () => true };
}
```

这段只展示替换工厂，不能单独当成完整扩展。实际实现还拦截 chunk 全局变量的赋值、webpack 对队列 `push` 的重设，并转交其他模块和 runtime 回调。源码匹配检查包括导出结构、Date/Function 检测及事件标记；不能只看模块编号就替换。

公共脚本的替代方法保留了页面初始化回调：

```js
// 仅在方法源码核验通过、原方法尚未执行时替换
sign.encryptPwd = function () {
  try {
    window.Detail?.initDg();
  } catch { /* 保留原方法捕获回调异常的行为 */ }
};
```

不能简单换成空函数：`Detail.initDg()` 会设置后续业务检查需要的状态。其他登录加密方法没有被改动。

### 验证时检查什么

| 检查 | 本次结果 |
| --- | --- |
| 两个替代入口是否实际调用 | 都已调用 |
| 页面是否仍主动刷新 | 观察期间没有 |
| 预览、编辑、保存是否正常 | 正常 |
| 刷新后内容是否保留 | 回读成功 |

必须在页面初始化前拦截。webpack 已注册模块后，再修改旧 chunk 队列，不能撤销已运行的检测。

## 3. 怎么复用到自己的项目

如果要实现同类前端检测，可以直接接入 `disable-devtool`。它支持上述 DOM、Date、Function 等探针；这里采用相同机制，不表示 BOSS 一定使用了这个库。

下面以现有 Vite 项目为例：检测命中后暂停页面提交并显示提示，关闭调试工具后恢复。保留用户输入，避免刷新循环。

### 安装

```sh
npm install disable-devtool
```

### 新增 `src/devtool-guard.js`

```js
import DisableDevtool from 'disable-devtool';

let blocked = false;
export const isPageBlocked = () => blocked;

function setBlocked(value) {
  blocked = value;
  window.dispatchEvent(new CustomEvent('devtool-state', {
    detail: { blocked: value }
  }));
}

export function startDevtoolGuard() {
  if (!import.meta.env.PROD) return; // 开发环境保留正常调试

  const result = DisableDevtool({
    detectors: [1, 3, 4], // DOM getter、Date、Function
    interval: 500,
    disableMenu: false,
    disableSelect: false,
    disableCopy: false,
    disableCut: false,
    disablePaste: false,
    clearLog: false,
    ondevtoolopen() { setBlocked(true); },
    ondevtoolclose() { setBlocked(false); }
  });

  if (!result.success) console.warn(result.reason);
}
```

使用自定义 `ondevtoolopen`，不调用库提供的关闭窗口动作。探针编号与参数含义见[库的配置说明](https://github.com/theajack/disable-devtool#312-parameters)。

### 在入口初始化，然后加载业务

```js
// src/main.js
import { startDevtoolGuard } from './devtool-guard.js';

startDevtoolGuard();
await import('./app.js');
```

把初始化放在入口，覆盖 SPA 的所有路由。不要在每个页面组件中重复启动检测。

### 在业务提交处接入

页面已有表单时，增加提示和提交检查：

```html
<p id="debug-warning" role="alert" hidden>
  检测到调试环境，已暂停提交。请关闭调试工具后重试。
</p>
<form id="editor">
  <textarea name="content"></textarea>
  <button type="submit">保存</button>
</form>
```

```js
// src/app.js
import { isPageBlocked } from './devtool-guard.js';

const form = document.querySelector('#editor');
const warning = document.querySelector('#debug-warning');
const button = form.querySelector('button[type="submit"]');

function updateState() {
  warning.hidden = !isPageBlocked();
  button.disabled = isPageBlocked();
}
window.addEventListener('devtool-state', updateState);
updateState(); // 初始化前已触发检测时，也能读到当前状态

form.addEventListener('submit', async event => {
  event.preventDefault();
  if (isPageBlocked()) return;
  // 在这里调用项目原有的保存方法
});
```

React/Vue 项目用组件状态控制提示和按钮，提交函数同样检查 `isPageBlocked()`；机制不变。

### 测试

```sh
npm run build
npm run preview
```

| 操作 | 要检查的结果 |
| --- | --- |
| 不开调试工具 | 能编辑和提交 |
| 打开调试工具并触发探针 | 显示提示，提交被暂停 |
| 关闭调试工具 | 能恢复提交，输入内容保留 |
| 切换 SPA 路由 | 检测仍在运行，没有重复初始化 |
| 慢设备、不同浏览器、辅助工具 | 是否误判；据结果调整启用的探针 |

接入代码是示例，未在你的项目中运行验收。它阻挡的是部分依赖调试工具的操作，不能可靠识别恶意 AI；本次插件就说明，这类前端入口仍能被提前替换。

## 参考

- [Chrome content scripts：注入时间与执行环境](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)
- [disable-devtool：用法、回调与检测类型](https://github.com/theajack/disable-devtool)
