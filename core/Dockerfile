# 构建阶段：解决依赖安装的缓存和锁文件问题
FROM node:20-alpine AS builder

WORKDIR /app

# 启用 corepack 并设置时区（避免构建时时区相关问题）
RUN corepack enable && \
    apk add --no-cache tzdata && \
    ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# 复制 workspace 配置文件（先复制锁文件利用构建缓存）
COPY pnpm-workspace.yaml pnpm-lock.yaml package.json ./
COPY core/package.json ./core/
COPY web/package.json ./web/

# 安装依赖：清理 PNPM 缓存 + 禁用严格锁文件（解决版本不匹配） + 国内源
RUN npm config set registry https://registry.npmmirror.com/ && \
    pnpm store prune && \
    pnpm install --no-frozen-lockfile

# 复制源代码（依赖安装完成后再复制，避免代码改动触发依赖重装）
COPY core ./core
COPY web ./web

# 构建前端项目
RUN pnpm build:web

# ==========================================
# 生产依赖阶段：仅安装 core 项目的运行时依赖
FROM node:20-alpine AS prod-deps

WORKDIR /app

RUN corepack enable && \
    apk add --no-cache tzdata && \
    ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# 复制必要的配置文件
COPY pnpm-workspace.yaml pnpm-lock.yaml package.json ./
COPY core/package.json ./core/

# 安装生产依赖：同样清理缓存 + 禁用严格锁文件
RUN npm config set registry https://registry.npmmirror.com/ && \
    pnpm store prune && \
    pnpm -C core install --prod --no-frozen-lockfile

# ==========================================
# 运行阶段：精简镜像，仅保留运行所需文件
FROM node:20-alpine AS runner

WORKDIR /app/core

# 设置生产环境和时区
ENV NODE_ENV=production
ENV TZ=Asia/Shanghai

# 安装必要的系统依赖（如时区、字体等，按需添加）
RUN apk add --no-cache tzdata && \
    ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    # 清理 apk 缓存，减小镜像体积
    rm -rf /var/cache/apk/*

# 复制生产依赖
COPY --from=prod-deps /app/node_modules ../node_modules/
COPY --from=prod-deps /app/core/node_modules ./node_modules/
COPY --from=prod-deps /app/package.json ../package.json

# 复制核心代码和构建好的前端资源
COPY --from=builder /app/core ./
COPY --from=builder /app/web/dist ../web/dist

# 暴露端口
EXPOSE 3000

# 非 root 用户运行（安全最佳实践）
RUN addgroup -g 1001 -S nodejs && \
    adduser -S app -u 1001 && \
    chown -R app:nodejs /app
USER app

# 启动命令
CMD ["node", "client.js"]