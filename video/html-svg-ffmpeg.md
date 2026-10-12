# 用 HTML/SVG 和 FFmpeg 制作视频

HTML/SVG 负责绘制画面和动画，浏览器将动画渲染为连续图像帧，FFmpeg 再把图像帧编码为视频。这个流程适合卡通、文字动效、图表和其他由网页元素组成的画面。

```mermaid
flowchart LR
    A[HTML / SVG 画面与动画] --> B[浏览器按时间渲染图像帧]
    B --> C[FFmpeg 编码]
    C --> D[MP4 视频]
```

## 制作步骤

### 1. 在 HTML/SVG 中绘制并定义动画

将画面绘制为 HTML 元素或内嵌 SVG，并提供一个按时间渲染的函数，例如 `window.renderAt(t)`。函数根据时间 `t` 计算角色位置、动作和其他动画属性。

按时间渲染可以让每一帧的画面可重复，帧率和时长也容易控制。

### 2. 用浏览器逐帧渲染

通过 Playwright 控制浏览器，在每个时间点调用 `renderAt(t)`，然后保存当前画面。以 24 fps、4 秒为例，需要生成 96 帧：

```javascript
const fps = 24;
const frameCount = fps * 4;

await page.setViewportSize({ width: 1280, height: 720 });

for (let i = 0; i < frameCount; i++) {
  await page.evaluate((t) => window.renderAt(t), i / fps);
  await page.screenshot({
    path: `frames/frame_${String(i).padStart(4, "0")}.png`,
  });
}
```

### 3. 用 FFmpeg 编码为 MP4

将连续编号的 PNG 帧按相同帧率交给 FFmpeg：

```bash
ffmpeg -framerate 24 -start_number 0 \
  -i frames/frame_%04d.png \
  -frames:v 96 -an \
  -c:v libx264 -crf 18 \
  -pix_fmt yuv420p -movflags +faststart \
  output/video.mp4
```

这里使用 H.264 编码且不包含音轨。调整帧率、帧数和输出参数，即可改变视频时长、画质和文件大小。

## 本次示例

示例用内嵌 SVG 绘制并动画化一只奔跑的小狗，再由浏览器输出图像帧并交给 FFmpeg 编码。

| 项目 | 结果 |
| --- | --- |
| 分辨率 | 1280×720 |
| 帧率与时长 | 24 fps，4 秒 |
| 总帧数 | 96 |
| 视频编码 | H.264，yuv420p |
| 文件大小 | 350,419 字节（约 342 KiB） |
| 音轨 | 无 |

## 依赖

- FFmpeg
- Chromium 浏览器及 Playwright

## 参考资料

- [FFmpeg 命令行文档](https://ffmpeg.org/ffmpeg.html)
- [FFmpeg image2 图像序列输入](https://ffmpeg.org/ffmpeg-formats.html#image2-1)
- [Playwright 页面截图文档](https://playwright.dev/docs/screenshots)
