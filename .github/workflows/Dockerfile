# 阶段 1: 基础镜像 (用于最终运行)
FROM alpine:3.20 AS base

# 阶段 2: 构建镜像 (安装 Rust 并编译)
FROM rust:1.75-alpine AS builder

# 安装编译所需的依赖 (musl-dev 用于静态链接，gcc 有时需要)
RUN apk add --no-cache musl-dev gcc git

# 设置工作目录
WORKDIR /app

# 复制 Cargo 配置文件先行，利用 Docker 缓存层
COPY Cargo.toml Cargo.lock* ./

# 创建 src 目录占位，防止后续 COPY 失败（如果 src 不存在）
RUN mkdir -p src

# 复制源代码
COPY . .

# 执行编译
# EasyTier 通常编译出的二进制名为 easytier-core
# 使用 --release 进行优化编译
# 目标架构通常由 Rust 自动识别，但在 Alpine musl 环境下可能需要指定 target
# 这里我们尝试直接编译，Rust 在 alpine 上默认会生成 musl 静态二进制
RUN cargo build --release

# 查找并复制编译产物
# 编译产物通常在 target/release/easytier-core
# 为了确保稳健，我们创建一个输出目录并复制文件
RUN mkdir -p /tmp/output && \
    cp target/release/easytier-core /tmp/output/easytier-core

# 阶段 3: 最终镜像
FROM base

# 安装运行时依赖 (时区数据和 tini 进程管理)
RUN apk add --no-cache tzdata tini

WORKDIR /app

# 从 builder 阶段复制编译好的二进制文件
COPY --from=builder --chmod=755 /tmp/output/easytier-core /usr/local/bin/easytier-core

# 设置时区环境变量
ENV TZ=Asia/Shanghai

# 暴露端口
# tcp
EXPOSE 11010/tcp
# udp
EXPOSE 11010/udp
# wg
EXPOSE 11011/udp
# ws
EXPOSE 11011/tcp
# wss
EXPOSE 11012/tcp

# 启动命令
ENTRYPOINT ["/sbin/tini", "--", "easytier-core"]
