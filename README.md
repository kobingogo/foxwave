# Foxwave · 狐狸电波 🦊🎧

一只戴着耳机的 3D 狐狸，陪你听音乐、发呆，或者专心做点事情。

Foxwave 是一个无需注册的浏览器音乐小玩具。音乐由 Web Audio 实时合成，狐狸会跟着节拍点头、摇尾巴；点点它、摸摸头，还能发现隐藏反应。

## 在线体验

[打开狐狸电波](https://foxwave.pages.dev)

## 三种心情，三个电台

| 电台 | 风格 | 节奏 | 氛围 |
| --- | --- | --- | --- |
| 88.1 | 深夜 Lo-fi | 74 BPM | 夜蓝色，适合专注和放空 |
| 101.7 | 霓虹 City Pop | 112 BPM | 日落粉，带一点复古公路感 |
| 96.3 | 森林午后 Bossa | 92 BPM | 森林绿，轻松的午后 |

频率是主题标识，音乐在浏览器中生成，并非真实广播流。

## 怎么玩

- 点击「打开 FOXWAVE」开始播放；点击耳机或底部电台按钮换台。
- 拖动旋转视角，滚轮缩放；在头顶来回轻抚，看看狐狸的反应。
- 鼻子、眼睛、嘴巴、耳朵、尾巴都有互动，页面内共有 8 个彩蛋。
- 右侧按钮可直接触发表情；暂停后放着一会儿，狐狸会睡着。
- 点击录制，可选择 10 / 30 / 60 秒或自由录制（最长 140 秒），生成 1080 × 1350 的视频，包含合成音乐。浏览器支持时导出 MP4，否则导出 WebM。

| 快捷键 | 功能 |
| --- | --- |
| 空格 | 播放 / 暂停 |
| ← / → 或 1 / 2 / 3 | 换台 |
| B | Boop |
| H | 显示 / 隐藏界面 |
| R | 开始 / 停止录制 |
| Esc | 关闭录制菜单或预览 |

彩蛋、Boop 次数和偏好保存在当前浏览器的 localStorage 中。

## 本地运行

无需安装项目依赖或构建。将仓库下载到本地后，在目录中启动静态服务器：

```sh
python3 -m http.server 8000
```

打开 http://localhost:8000 。请通过 HTTP 访问，直接双击 HTML 可能无法加载模型。

## 项目结构

- `index.html`：界面、3D 渲染、交互、音乐合成和视频录制。
- `fox-web.glb`：狐狸模型。
- `.gitignore`：排除本地复盘、旧版页面和临时文件。

采用 Three.js 0.186.1、Web Audio、WebGL 与 MediaRecorder；Three.js / Draco 从 jsDelivr 加载，字体来自 Google Fonts。需要能够访问这些外部服务，并使用支持 WebGL 的现代浏览器。录制格式取决于浏览器，移动端表现也受设备性能影响。

## 部署到 Cloudflare Pages

项目已连接 GitHub，推送到 `main` 分支后会自动构建并发布。

Cloudflare Pages 设置：

- 仓库：`kobingogo/foxwave`
- 生产分支：`main`
- 构建命令：`mkdir -p dist && cp index.html fox-web.glb dist/`
- 输出目录：`dist`

发布目录只包含 `index.html` 和 `fox-web.glb`，无需安装项目依赖。

部署说明：[Cloudflare Pages Git 集成](https://developers.cloudflare.com/pages/configuration/git-integration/)。

`FOX-FM-技术复盘.md` 和 `index.v1.html` 仅保留在本地，不进入 GitHub 或网站部署。
