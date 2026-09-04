# 在 macOS 上作为本机 xray 客户端面板运行

本文档记录如何把本面板（3x-ui fork）改造为 macOS 本机使用的 xray 客户端管理面板：
只监听 `127.0.0.1` 单端口、纯 HTTP、不申请证书、不监听公网。**无需修改任何代码**，
全部为构建与配置操作。以下步骤在 Apple Silicon (arm64) Mac 上验证通过。

## 前提

- Go 1.27+
- Node.js 24（见 `.nvmrc`）+ npm 10+
- Xcode Command Line Tools（CGO SQLite 驱动需要 clang）：

  ```bash
  xcode-select --install
  ```

## 1. 构建前端 + 面板二进制

必须在 Mac 本机编译（SQLite 驱动依赖 CGO，交叉编译不可用）：

```bash
cd frontend && npm install && npm run build && cd ..
go build -o x-ui .
```

注意：裸 `go build` 的产物以目录名命名（`3x-ui`），用 `-o x-ui` 显式指定。

## 2. 配置本地数据目录

默认数据目录是 `/etc/x-ui`，直接启动会报
`Database initialization failed: mkdir /etc/x-ui: permission denied`。
用 `.env` 把数据库、日志、xray 二进制全部收进项目内目录：

```bash
cp .env.example .env
mkdir -p x-ui
```

也可以改用环境变量指向任意目录（`XUI_DB_FOLDER` / `XUI_LOG_FOLDER` / `XUI_BIN_FOLDER`）。

## 3. 放置 xray 二进制（本机踩过的坑）

xray 二进制需放在 `XUI_BIN_FOLDER` 下，命名为 `xray-darwin-arm64`（Intel Mac 为
`xray-darwin-amd64`）。两个要点：

- **必须是真实文件，不能是符号链接**。符号链接会导致面板重启 xray 时报
  `fork/exec ...: no such file or directory`（假 ENOENT，链接目标不可达所致）。
  直接复制二进制实体：

  ```bash
  cp /path/to/xray x-ui/bin/xray-darwin-arm64   # 不要用 ln -s
  chmod +x x-ui/bin/xray-darwin-arm64
  xattr -d com.apple.quarantine x-ui/bin/xray-darwin-arm64  # 浏览器下载的需去隔离属性
  ```

- 验证：`file` 应显示 `Mach-O 64-bit executable arm64`，且终端里能直接跑
  `./x-ui/bin/xray-darwin-arm64 -version`。

更省事的替代方案：不放二进制，启动后用面板内置的 xray 版本切换功能自行下载
（原生支持 darwin，自动拉取官方 `Xray-macos-arm64.zip` 并校验哈希）。

## 4. 启动与初始化

```bash
./x-ui run
```

- 全新数据库的默认账号：`admin` / `admin`（首次登录后立即修改）。
- 默认地址：`http://127.0.0.1:2053/`。

## 5. 安全收敛：只监听回环 + 自定义路径

一条命令完成（也可在面板设置页逐项修改，改完重启生效）：

```bash
./x-ui setting -listenIP 127.0.0.1 -webBasePath mypanel -username 新用户名 -password 新密码
```

- `webBasePath` 会自动补全为 `/mypanel/`；设置后原路径即失效，面板地址变为
  `http://127.0.0.1:2053/mypanel/`，务必记牢。忘记时可用 `setting` CLI 重设。
- 证书相关无需任何操作：本项目没有自动申请证书的逻辑，`webCertFile`/`webKeyFile`
  为空（默认）即纯 HTTP，HSTS 与 session cookie 的 Secure 标志自动关闭。
- 订阅子服务（独立端口 2096）如不需要，在设置里关闭 `subEnable` 即可。

## 6. 本机代理用法

新建一个监听 `127.0.0.1:1080` 的 **mixed** inbound（同一端口同时提供 SOCKS5 和
HTTP 代理，即 ClashX/v2rayN 的 mixed 端口概念），把 macOS 系统/浏览器代理指向它即可。

## 已知功能降级（macOS 上不可用）

- **IP 限速/封禁（fail2ban）**：静默不生效，仅记录观察到的 IP。
- **面板内一键更新**：仅限 Linux，升级需手动重新执行第 1 步构建。
- **MTProto inbound**：mtg-multi 官方无 darwin 预编译二进制，需自行编译。
- **系统日志查看**：依赖 journalctl，macOS 上不可用。
