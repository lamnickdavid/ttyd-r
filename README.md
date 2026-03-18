# ttyd 定制版

这是一个基于原版 `ttyd` 修改的私有定制版本，用于个人网页终端部署。

## 上游项目

- 原项目：`https://github.com/tsl0922/ttyd`
- 上游版本基线：`1.7.7`
- 原项目许可证：`MIT`

本仓库不是官方上游仓库，而是在上游 `ttyd` 基础上做了定制补丁，主要用于当前这套局域网 + FRP + Nginx 的部署场景。

## 主要改动

- 修复移动端浏览器在启用 `-c username:password` 时 WebSocket 一直 `reconnect` 的问题
- 保留页面级 Basic Auth 登录，同时让 `/ws` 阶段改走 `AuthToken`
- 给前端启用 xterm `ClipboardAddon`
- 支持 `opencode` 这类 TUI 程序通过终端剪贴板协议把内容同步到浏览器系统剪贴板
- 保留当前已经验证可用的网页终端部署方式

## 当前用途

- 局域网访问 `ttyd`
- 通过 `frpc` / `frps` / Nginx 暴露到公网域名
- 浏览器打开网页时弹出用户名密码框
- 支持 PC、iPhone、安卓浏览器访问

## 仓库说明

建议这个仓库只保存源码和必要文件，不提交构建产物依赖目录。

建议保留：

- `src/`
- `html/src/`
- `html/package.json`
- `html/yarn.lock`
- `html/.yarnrc.yml`
- `src/html.h`
- `CMakeLists.txt`
- `README.md`

建议忽略：

- `html/dist/`
- `html/node_modules/`
- `html/.yarn/`
- `build/`
- `alpine-build/`
- `alpine-static/`
- `static-prefix/`
- 其他临时构建目录

## 构建说明

这个版本包含两类定制：

- 后端 C 代码补丁
- 前端 xterm 剪贴板补丁

如果修改了前端代码，需要先重新生成 `src/html.h`，再重新编译 `ttyd` 二进制。

前端构建：

```bash
cd html
corepack yarn install
corepack yarn build
```

后端编译和静态构建方式请参考部署文档。

## Release 建议

Gitea 仓库保存源码，Release 附件保存编译后的成品二进制：

- Release 文件：`ttyd`
- 可选附带：`DEPLOYMENT.md`
- 可选附带：`SHA256SUMS`

建议 Release 名称示例：

- `ttyd-1.7.7-mobile-clipboard-patched`

## 部署文档

完整部署步骤、补丁说明、编译环境、Nginx / FRP / systemd 配置，请参考发布包里的 `DEPLOYMENT.md`。

## 说明

- 这个版本适合当前个人环境使用
- 不建议把明文凭据提交到仓库
- 当前网页终端本质上仍然是可写 Shell，请谨慎暴露到公网
