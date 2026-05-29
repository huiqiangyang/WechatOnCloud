# WechatOnCloud

在飞牛 NAS（x86_64）上运行服务端微信，多个 web 用户通过浏览器访问**同一个**微信会话，实现跨设备消息同步、多端共享。

> 设计与选型详见 [技术方案.md](技术方案.md)。
> 本仓库当前是 **MVP**：Docker + 官方微信 Linux 原生版 + KasmVNC 串流到浏览器。

---

## 工作原理（一句话）

容器里跑 Xvfb 虚拟显示 + 官方原版微信，KasmVNC 把画面串到浏览器。多个浏览器连同一容器 = 共享同一个微信会话。**不修改微信客户端**。

---

## 快速开始（零配置）

> 需已安装 Docker（含 Compose 插件）。x86_64 / arm64 均可。

```bash
git clone <this-repo> WechatOnCloud
cd WechatOnCloud
docker compose up -d --build
```

就这一条。然后浏览器访问 `http://<NAS_IP>:36080`（或 `https://<NAS_IP>:36443` 自签证书），
用默认账号 **admin / wechat** 登录 web 端，再用手机扫码登录微信即可。

> 宿主端口默认用冷门的 `36080/36443` 避免与其他服务冲突；容器内仍是 3000/3001。要改见 `.env`。

> 首次构建会下载约 190~210MB 的微信安装包，较慢，属正常。

### 架构自动适配

构建时会**自动检测本机 CPU 架构**（Docker BuildKit 的 `TARGETARCH`），下载对应的官方微信包：

| 构建机器 | 架构 | 自动下载 |
|----------|------|----------|
| Intel/AMD NAS、x86 服务器 | amd64 | `WeChatLinux_x86_64.deb` |
| ARM NAS、Apple Silicon Mac | arm64 | `WeChatLinux_arm64.deb` |

所以**在你的 Mac（M 系列）上直接 `docker compose up -d --build` 就能本地调试**，到飞牛上（无论 x64 还是 arm）同样一条命令，无需改任何架构相关配置。

> 仅当你要在 A 架构机器上为 B 架构 NAS **交叉构建**时，才需在 `docker-compose.yml` 里取消 `platforms` 注释并指定目标架构。

### 自定义配置（可选）

默认值开箱即用，无需任何配置。要改的话，复制一份 `.env`：

```bash
cp .env.example .env   # 然后按需修改，最该改的是 WOC_PASSWORD
```

可配置项见 [.env.example](.env.example)：web 账号密码、PUID/PGID、时区、端口。

---

## 数据持久化

微信登录态与消息都写在容器内 `HOME=/config`，已映射到宿主 `./data`。
- **不要删 `./data`**，否则要重新扫码登录、丢本地消息缓存。
- 备份微信 = 备份 `./data` 目录。

---

## ⚠️ 安全须知（必读）

这套系统暴露的是一个**已登录的微信**——能访问端口的人就能看你聊天记录、以你身份发消息。

MVP 仅依赖 KasmVNC 自带的 web 端基础鉴权（`CUSTOM_USER`/`PASSWORD`）。**生产使用务必：**

- 只在内网访问，或经飞牛远程访问 / VPN / 内网穿透，**不要直接裸暴露公网**；
- 务必改掉默认密码（默认 admin / wechat）：`cp .env.example .env` 后改 `WOC_PASSWORD`；
- 进一步加固（独立鉴权层 Authelia、反代 TLS、陌生设备验证码）见 [技术方案.md](技术方案.md) 第 5 节，属后续迭代。

---

## 常见问题

| 现象 | 排查 |
|------|------|
| 界面/消息显示成方块 | 中文字体没装好，确认镜像构建时 `fonts-noto-cjk` 安装成功 |
| 微信起不来 / 黑屏 | 看日志 `docker logs wechat-on-cloud`；确认 `security_opt: seccomp:unconfined` 与 `shm_size` 已生效。微信 deb 漏声明的运行时依赖（libatomic1、xcb 系、GTK3、Chromium 所需 X 扩展等）已在 Dockerfile 内置 |
| 排查缺哪个库 | `docker exec wechat-on-cloud ldd /opt/wechat/wechat` 及 `.../RadiumWMPF/runtime/WeChatAppEx`，看 `not found` 的项，补进 Dockerfile 的依赖层 |
| 多人同时操作很乱 | 已知问题：单会话多端共享、键鼠会打架。MVP 未做并发控制，建议同一时刻一人操作（见技术方案 6.1） |
| 过段时间掉登录 | 微信桌面会话会定期失效，需手机重新扫码（见技术方案 6.2） |
| 下载 deb 失败 | 腾讯 CDN 偶发波动，重试 `docker compose build --no-cache`；Dockerfile 已内置主/备 CDN 自动回退 |
| 架构不支持报错 | 微信仅提供 x86_64 / arm64；若构建机是其他架构会直接报错退出 |

查看运行日志：

```bash
docker logs -f wechat-on-cloud
```

---

## 目录结构

```
WechatOnCloud/
├── docker/
│   ├── Dockerfile      # KasmVNC base + 中文字体 + 按架构自动下载微信原生版 deb
│   └── autostart       # openbox 会话启动时拉起微信（含崩溃自重启）
├── docker-compose.yml  # 端口 / 数据卷 / seccomp / 鉴权（缺省值开箱即用）
├── .env.example        # 可选配置项（账号密码、PUID/PGID、端口、时区）
├── 技术方案.md          # 完整设计文档
└── README.md
```

---

## 路线图

- [x] MVP：Docker + 微信原生版 + KasmVNC，浏览器扫码登录、收发消息
- [ ] 反代 + 独立鉴权 + TLS 加固
- [ ] 多端并发控制（控制权令牌）
- [ ] 掉登录时 web 端二维码重扫入口
- [ ] 换 Xpra 做"单窗口独立 App"观感
- [ ] 打包成飞牛原生 fpk 分发
