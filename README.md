# 佩奇拼图 PWA

这是一个可直接部署到 GitHub Pages 的静态 PWA 小游戏。

## 本地文件

- `index.html`：游戏页面
- `manifest.webmanifest`：PWA 安装信息
- `sw.js`：离线缓存
- `icons/`：主屏幕图标
- `peppa1.jpg` 到 `peppa9.jpg`：9 个关卡图片

游戏支持在浏览器里导入新图片。导入图片只会保存在当前设备的浏览器本地存储中，不会上传到服务器。用户也可以删除当前导入图，或清空全部导入图。

## GitHub Pages 部署

1. 在 GitHub 新建一个仓库，例如 `peppa-puzzle`。
2. 把本文件夹里的所有文件上传到仓库根目录。
3. 进入仓库 `Settings` -> `Pages`。
4. `Build and deployment` 选择 `Deploy from a branch`。
5. Branch 选择 `main`，目录选择 `/ (root)`，保存。
6. 等待 GitHub Pages 部署完成后，打开页面给出的访问地址。

## iPhone / iPad 安装

1. 用 Safari 打开 GitHub Pages 地址。
2. 点分享按钮。
3. 选择“添加到主屏幕”。
4. 从主屏幕打开后会像 App 一样运行。
