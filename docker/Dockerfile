# WechatOnCloud —— 在浏览器里访问服务端微信
# 基于 linuxserver 的 KasmVNC 基础镜像（多架构 amd64/arm64）：
# 自带 Xvfb + openbox + KasmVNC + web 客户端(3000/3001)
FROM lscr.io/linuxserver/baseimage-kasmvnc:debianbookworm

# TARGETARCH 由 Docker BuildKit 自动注入：本机构建=本机架构，
# 因此在 x86 NAS / arm NAS / Apple Silicon Mac 上 `docker compose build` 都会自动选对包。
ARG TARGETARCH
# CDN 主链失败时回退备链
ARG WECHAT_CDN="https://dldir1v6.qq.com/weixin/Universal/Linux"
ARG WECHAT_CDN_FALLBACK="https://dldir1.qq.com/weixin/Universal/Linux"

ENV DEBIAN_FRONTEND=noninteractive

RUN set -eux; \
    # 1) 按架构映射微信 deb 文件名
    case "${TARGETARCH}" in \
        amd64) WECHAT_FILE="WeChatLinux_x86_64.deb" ;; \
        arm64) WECHAT_FILE="WeChatLinux_arm64.deb" ;; \
        *) echo "不支持的架构: ${TARGETARCH}（微信仅提供 x86_64 / arm64）" >&2; exit 1 ;; \
    esac; \
    echo "构建架构=${TARGETARCH}, 微信包=${WECHAT_FILE}"; \
    # 2) 依赖：中文字体（否则界面/消息显示方块）+ 语言环境 + 下载工具
    apt-get update; \
    apt-get install -y --no-install-recommends \
        curl ca-certificates locales \
        fonts-wqy-zenhei fonts-wqy-microhei fonts-noto-cjk fonts-noto-color-emoji \
        libnss3 libgbm1 libasound2 libxss1; \
    sed -i 's/# zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen; \
    locale-gen; \
    # 3) 下载微信 deb（主链失败回退备链）
    ( curl -fSL --retry 3 -A "Mozilla/5.0" -o /tmp/wechat.deb "${WECHAT_CDN}/${WECHAT_FILE}" \
      || curl -fSL --retry 3 -A "Mozilla/5.0" -o /tmp/wechat.deb "${WECHAT_CDN_FALLBACK}/${WECHAT_FILE}" ); \
    # 4) 用 apt 安装本地 deb 以自动解决依赖
    apt-get install -y --no-install-recommends /tmp/wechat.deb; \
    rm -f /tmp/wechat.deb; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*

# 微信 deb 未声明、但运行时需要的额外库（单独成层，避免动到上面缓存的下载层）。
# 微信原生版是 Qt 程序，依赖一组 xcb 平台库；libxcb-cursor0 由 Qt 动态 dlopen，ldd 查不到，需主动装。
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libatomic1 \
        libxdamage1 \
        libxkbcommon-x11-0 \
        libxcb-icccm4 \
        libxcb-image0 \
        libxcb-keysyms1 \
        libxcb-render-util0 \
        libxcb-xkb1 \
        libxcb-cursor0 \
        # WeChatAppEx 是 Chromium 内核，需 GTK3 全家桶 + 一组 X 扩展 + cups
        libgtk-3-0 \
        libatk1.0-0 \
        libatk-bridge2.0-0 \
        libatspi2.0-0 \
        libcups2 \
        libxcomposite1 \
        libxrandr2 \
        libxfixes3 \
        libxtst6 \
        libxshmfence1 \
        libdrm2; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*

ENV LANG=zh_CN.UTF-8 \
    LC_ALL=zh_CN.UTF-8 \
    LIBGL_ALWAYS_SOFTWARE=1

# openbox 会话启动时执行此脚本拉起微信
COPY autostart /defaults/autostart
RUN chmod +x /defaults/autostart

# 3000 = HTTP web 客户端, 3001 = HTTPS
EXPOSE 3000 3001
