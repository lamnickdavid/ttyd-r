# ttyd 部署文档

本文记录 `ttyd` 的完整部署过程，包括本机安装、`systemd` 服务、`frpc` 转发、公网 Nginx 反向代理，以及移动端兼容补丁版的构建方式。

## 1. 部署目标

- 局域网访问：`http://192.168.110.218:7681`
- 认证方式：浏览器打开页面时弹出用户名密码框

## 2. 本机环境信息

- ttyd 二进制：`/ProgramFiles/ttyd/ttyd`
- ttyd 源码目录：`/ProgramFiles/ttyd-r`
- ttyd 服务文件：`/.config/systemd/user/ttyd.service`
- ttyd 凭据文件：`/.config/ttyd/ttyd.env`

## 3. 下载并安装官方 ttyd

先安装官方发布版二进制：

```bash
mkdir -p /ProgramFiles/ttyd
cd ProgramFiles/ttyd
wget https://github.com/tsl0922/ttyd/releases/latest/download/ttyd.x86_64 -O ttyd
chmod +x ttyd
./ttyd --version
```

说明：

- 官方 `1.7.7` 对 PC 浏览器通常没问题。
- 但在启用 `-c username:password` 后，iPhone 和部分安卓浏览器可能会一直显示 `reconnect`。
- 原因是移动端浏览器的 WebSocket 请求不一定会带上 Basic Auth header。
- 因此当前正式部署实际使用的是“补丁版静态二进制”。

## 4. 构建移动端兼容的补丁版 ttyd

### 4.1 拉取源码

```bash
git clone --branch 1.7.7 --depth 1 https://github.com/tsl0922/ttyd.git /ProgramFiles/ttyd-src
```

### 4.2 修改源码补丁

编辑 `/ProgramFiles/ttyd-src/src/protocol.c`，让 `/ws` 握手不再强依赖 Basic Auth header。

关键逻辑应为：

```c
      n = lws_hdr_copy(wsi, pss->path, sizeof(pss->path), WSI_TOKEN_GET_URI);
#if defined(LWS_ROLE_H2)
      if (n <= 0) n = lws_hdr_copy(wsi, pss->path, sizeof(pss->path), WSI_TOKEN_HTTP_COLON_PATH);
#endif
      if (strncmp(pss->path, endpoints.ws, n) != 0) {
        lwsl_warn("refuse to serve WS client for illegal ws path: %s\n", pss->path);
        return 1;
      }

      if (server->credential == NULL || server->auth_header != NULL) {
        if (!check_auth(wsi, pss)) return 1;
      }
```

这段补丁的含义是：

- 页面本身仍然需要浏览器 Basic Auth 登录
- WebSocket 阶段不再重复要求 Basic Auth header
- WebSocket 改为依赖 `ttyd` 自己的 `/token` + `AuthToken` 机制
- 这样移动端浏览器也能正常建立终端连接

另外，为了让 `opencode` 这类 TUI 程序通过终端剪贴板协议把内容真正写入浏览器系统剪贴板，还需要给前端启用 xterm 的 Clipboard addon。

需要修改的前端文件：

- `/ProgramFiles/ttyd-r/html/package.json`
- `/ProgramFiles/ttyd-r/html/src/components/terminal/xterm/index.ts`

关键改动：

- 增加依赖 `@xterm/addon-clipboard`
- 在前端 `Terminal` 初始化时加载 `ClipboardAddon`

这样处理后：

- 普通网页终端选中文本仍然可以直接复制
- `opencode` 内部显示“已复制到剪贴板”的操作也能真正同步到浏览器系统剪贴板
- `Ctrl+Shift+V` 可以粘贴刚才在 `opencode` 里复制的内容

### 4.3 在 Alpine 容器里构建静态二进制

本次构建使用的是比较基础的 Linux C 项目编译环境，不是复杂的交叉编译或大型工具链。

宿主机需要：

- `docker`
- `git`
- `systemctl`

Alpine 容器内安装的构建工具：

- `build-base`
- `cmake`
- `git`
- `curl`
- `pkgconf`
- `linux-headers`

Alpine 容器内安装的依赖库：

- `zlib-dev`
- `zlib-static`
- `json-c-dev`
- `libuv-dev`
- `libuv-static`
- `mbedtls2-dev`
- `mbedtls2-static`
- `libcap-dev`
- `libcap-static`

构建过程中还会额外获取：

- `ttyd 1.7.7` 源码
- `libwebsockets 4.3.3` 源码

整体流程是：

- 在 Alpine 容器里先编译静态 `libwebsockets`
- 再编译 `ttyd`
- 最后通过 `cc -static` 手动链接出可直接部署的静态二进制

说明：

- 这些属于常规 C 项目编译环境
- 构建完成后可以删除源码目录里的构建中间目录，不影响已经部署好的 `/ProgramFiles/ttyd/ttyd`

### 4.4 重新构建前端并生成新的 `html.h`

在 `/ProgramFiles/ttyd-src/html` 下执行：

```bash
corepack yarn install
corepack yarn build
```

这一步会：

- 安装前端依赖
- 重新打包浏览器端资源
- 通过 `gulp` 生成新的 `/ProgramFiles/ttyd-src/src/html.h`

`ttyd` 的网页前端资源最终是编译进二进制里的，所以如果改了前端代码但没有重新生成 `html.h`，运行中的 `ttyd` 页面不会生效。

### 4.5 在 Alpine 容器里编译静态二进制

执行下面的构建命令：

```bash
docker run --rm -v "/ProgramFiles/ttyd-src:/src" alpine:3.20 sh -lc 'apk add --no-cache build-base cmake git curl pkgconf zlib-dev zlib-static json-c-dev libuv-dev libuv-static mbedtls2-dev mbedtls2-static libcap-dev libcap-static linux-headers >/dev/null && git config --global --add safe.directory /src && rm -rf /src/static-prefix /src/alpine-static /tmp/lws-build /tmp/libwebsockets-4.3.3 /tmp/libwebsockets.tar.gz && curl -fL https://github.com/warmcat/libwebsockets/archive/refs/tags/v4.3.3.tar.gz -o /tmp/libwebsockets.tar.gz && tar -xzf /tmp/libwebsockets.tar.gz -C /tmp && cmake -S /tmp/libwebsockets-4.3.3 -B /tmp/lws-build -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/src/static-prefix -DCMAKE_FIND_LIBRARY_SUFFIXES=.a -DLWS_WITH_MBEDTLS=ON -DLWS_WITH_LIBUV=ON -DLWS_WITH_SHARED=OFF -DLWS_WITH_STATIC=ON -DLWS_WITH_EVLIB_PLUGINS=OFF -DLWS_WITHOUT_TESTAPPS=ON -DLWS_ROLE_RAW_FILE=OFF -DLWS_UNIX_SOCK=ON -DLWS_IPV6=ON -DLWS_WITH_HTTP2=ON -DLWS_WITH_HTTP_BASIC_AUTH=OFF -DLWS_WITH_UDP=OFF -DLWS_WITHOUT_CLIENT=ON -DLWS_WITHOUT_EXTENSIONS=OFF -DLWS_WITH_LEJP=OFF -DLWS_WITH_LEJP_CONF=OFF -DLWS_WITH_LWSAC=OFF -DLWS_WITH_SEQUENCER=OFF -DLWS_WITH_PLUGINS=OFF -DLWS_WITH_ZLIB=ON -DLWS_ZLIB_LIBRARIES=/lib/libz.a -DLWS_ZLIB_INCLUDE_DIRS=/usr/include -DLWS_MBEDTLS_LIBRARIES="/usr/lib/libmbedtls.a;/usr/lib/libmbedx509.a;/usr/lib/libmbedcrypto.a" -DLWS_MBEDTLS_INCLUDE_DIRS=/usr/include -DLIBUV_LIBRARIES=/usr/lib/libuv.a -DLIBUV_INCLUDE_DIRS=/usr/include && cmake --build /tmp/lws-build -j"$(nproc)" && cmake --install /tmp/lws-build && cmake -S /src -B /src/alpine-static -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/src/static-prefix -DCMAKE_FIND_LIBRARY_SUFFIXES=.a -DCMAKE_EXE_LINKER_FLAGS="-static -s" -DLIBUV_INCLUDE_DIR=/usr/include -DLIBUV_LIBRARY=/usr/lib/libuv.a -DJSON-C_INCLUDE_DIR=/usr/include/json-c -DJSON-C_LIBRARY=/usr/lib/libjson-c.a -DZLIB_INCLUDE_DIR=/usr/include -DZLIB_LIBRARY=/lib/libz.a -DZLIB_LIBRARY_RELEASE=/lib/libz.a -DZLIB_LIBRARIES=/lib/libz.a && cmake --build /src/alpine-static -j"$(nproc)" || true && cc -static -s /src/alpine-static/CMakeFiles/ttyd.dir/src/utils.c.o /src/alpine-static/CMakeFiles/ttyd.dir/src/pty.c.o /src/alpine-static/CMakeFiles/ttyd.dir/src/protocol.c.o /src/alpine-static/CMakeFiles/ttyd.dir/src/http.c.o /src/alpine-static/CMakeFiles/ttyd.dir/src/server.c.o -o /src/alpine-static/ttyd /src/static-prefix/lib/libwebsockets.a /usr/lib/libjson-c.a /usr/lib/libuv.a /usr/lib/libmbedtls.a /usr/lib/libmbedx509.a /usr/lib/libmbedcrypto.a /usr/lib/libcap.a /lib/libz.a -pthread -lm'
```

构建完成后，补丁版二进制位于：

- `/ProgramFiles/ttyd-src/alpine-static/ttyd`

### 4.6 部署补丁版二进制

```bash
install -m 755 /ProgramFiles/ttyd-src/alpine-static/ttyd /ProgramFiles/ttyd/ttyd
systemctl --user restart ttyd.service
```

## 5. 创建 ttyd 凭据文件

```bash
mkdir -p /.config/ttyd
chmod 700 /.config/ttyd
cat > /.config/ttyd/ttyd.env <<'EOF'
TTYD_CREDENTIAL=your_username:your_password
EOF
chmod 600 /.config/ttyd/ttyd.env
```

说明：

- 认证由 `ttyd` 本身完成，不是 Nginx 完成
- 浏览器打开页面时会先弹出用户名密码框
- 请将 `your_username:your_password` 替换成实际值

## 6. 创建 systemd 用户服务

创建 `/.config/systemd/user/ttyd.service`：

```ini
[Unit]
Description=ttyd web terminal service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/ProgramFiles
Environment=TERM=xterm-256color
EnvironmentFile=/.config/ttyd/ttyd.env
ExecStart=/bin/sh -lc 'exec /ProgramFiles/ttyd/ttyd --interface 0.0.0.0 --port 7681 --check-origin -c "$TTYD_CREDENTIAL" --writable /bin/bash -l'
Restart=on-failure
RestartSec=2

[Install]
WantedBy=default.target
```

启用并启动服务：

```bash
systemctl --user daemon-reload
systemctl --user enable --now ttyd.service
systemctl --user status ttyd.service
```

如果希望退出登录后服务仍然运行，可选执行：

```bash
sudo loginctl enable-linger user
```

## 7. 验证本机和局域网访问

未带账号密码：

```bash
curl -I http://127.0.0.1:7681
curl -I http://192.168.110.218:7681
```

预期：

- 返回 `401 Unauthorized`
- 响应头包含 `www-authenticate: Basic realm="ttyd"`

带账号密码：

```bash
curl -I -u 'your_username:your_password' http://127.0.0.1:7681
curl -I -u 'your_username:your_password' http://192.168.110.218:7681
```

预期：

- 返回 `200 OK`

验证桌面和移动端共用的 `/token` 机制：

```bash
curl -su 'your_username:your_password' http://127.0.0.1:7681/token
```

预期：

- 返回 `{"token":"..."}`

验证 WebSocket 握手在不带 Basic Auth header 时也能建立：

```bash
curl --http1.1 -is --max-time 10 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' \
  -H 'Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==' \
  -H 'Origin: http://127.0.0.1:7681' \
  http://127.0.0.1:7681/ws
```

预期：

- 返回 `101 Switching Protocols`

## 8. 配置本机 frpc

编辑 `/ProgramFiles/Frp/frpc.toml`，加入：

```toml
[[proxies]]
name = "ttyd"
type = "http"
localIP = "127.0.0.1"
localPort = 7681
customDomains = ["*********"]
```

如果文件里已经有其他服务，只需要追加这一段。

校验并重启：

```bash
/ProgramFiles/Frp/frpc verify -c /ProgramFiles/Frp/frpc.toml
sudo systemctl restart frpc.service
sudo systemctl status frpc.service
```

预期日志关键词：

- `proxy added: [ ... ttyd ]`
- `[ttyd] start proxy success`

## 9. 配置阿里云服务器上的 Nginx

公网服务器使用 Nginx 作为入口，网页应用统一走本机 `7080`。

证书文件：

- `/etc/letsencrypt/live/*****/fullchain.pem`
- `/etc/letsencrypt/live/*****/privkey.pem`

创建或修改 `*****` 的 Nginx 配置：

```nginx
server {
    listen 80;
    server_name *****;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name *****;

    ssl_certificate     /etc/letsencrypt/live/*****/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/*****/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:****;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}
```

重载 Nginx：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

注意：

- 不要再在 Nginx 里额外配置 `auth_basic`，否则会变成双重认证
- 页面请求和 WebSocket 请求都必须转发到 `127.0.0.1:7080`
- `Upgrade` 和 `Connection "upgrade"` 是必须项，否则页面能打开但终端连不上

## 10. 验证公网访问

不带凭据：

```bash
curl -kI https://*****
```

预期：

- 返回 `401 Unauthorized`

带凭据：

```bash
curl -kI -u 'your_username:your_password' https://*****
```

预期：

- 返回 `200 OK`

验证 `/token`：

```bash
curl -sku 'your_username:your_password' https://*****/token
```

预期：

- 返回 `{"token":"..."}`

验证公网 WebSocket：

```bash
curl --http1.1 -isk --max-time 15 \
  -H 'Connection: Upgrade' \
  -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' \
  -H 'Sec-WebSocket-Key: ******' \
  -H 'Origin: https://*****' \
  https://*****/ws
```

预期：

- 返回 `101 Switching Protocols`

浏览器验证：

1. 打开 `https://*****`
2. 浏览器弹出用户名密码框
3. 输入 `/.config/ttyd/ttyd.env` 中的账号密码
4. 认证通过后进入终端页面
5. PC、iPhone、安卓浏览器都可以正常连接

## 11. 日常管理命令

查看 ttyd：

```bash
systemctl --user status ttyd.service
journalctl --user -u ttyd.service -f
```

重启 ttyd：

```bash
systemctl --user restart ttyd.service
```

查看 frpc：

```bash
sudo systemctl status frpc.service
sudo journalctl -u frpc.service -f
```

重启 frpc：

```bash
sudo systemctl restart frpc.service
```

查看公网服务器 Nginx：

```bash
sudo nginx -t
sudo systemctl status nginx
sudo journalctl -u nginx -f
```

## 12. 当前实际部署状态

当前环境已经完成：

- `ttyd` 已安装在 `/ProgramFiles/ttyd`
- 当前运行的是从 `/ProgramFiles/ttyd-src` 编译出来的补丁版静态二进制
- 当前前端已经启用 xterm `ClipboardAddon`，用于支持 `opencode` 等 TUI 的浏览器剪贴板同步
- `ttyd` 用户级 systemd 服务已创建并启用
- `ttyd` 监听在 `0.0.0.0:7681`
- `frpc` 已包含 `*****` 的 HTTP 代理
- `*****` 证书已签发
- 阿里云 Nginx 反向代理和 WebSocket 转发已正常工作
- 本地 `/token` 和公网 `/token` 已正常返回 token
- 本地 `/ws` 和公网 `/ws` 已正常返回 `101 Switching Protocols`

## 13. 安全建议

- 公网开放时尽量使用强密码，不要长期使用弱口令
- 如果条件允许，可以在 Nginx 或防火墙层限制来源 IP
- 由于 `ttyd` 实际上提供的是一个可写的 `/bin/bash -l`，请把它视为高风险远程 Shell 服务
- 如果将来长期对公网开放，建议再加一层更强的上游认证机制
