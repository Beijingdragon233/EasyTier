# 阶段 1: 基础镜像 (用于最终运行)
FROM alpine:3.20 AS base

# 阶段 2: 构建镜像 (安装 Rust 并编译)
FROM rust:1.75-alpine AS builder

# 安装编译所需的依赖
# musl-dev: 用于生成静态链接二进制文件 (Alpine 必备)
# gcc: 某些 crate 可能需要 C 编译器
# git: Cargo 下载依赖时需要
RUN apk add --no-cache musl-dev gcc git

WORKDIR /app

# 1. 先复制根目录的 Cargo.toml 和 Cargo.lock (如果存在)
# 这有助于利用 Docker 缓存层，避免每次修改代码都重新下载依赖
COPY Cargo.toml Cargo.lock* ./

# 2. 复制各个成员 crate 的 Cargo.toml
# 根据截图，我们需要手动复制这些子目录的配置文件，以便 cargo 能解析 workspace
COPY easytier/Cargo.toml ./easytier/
COPY easytier-rpc-build/Cargo.toml ./easytier-rpc-build/
COPY easytier-web/Cargo.toml ./easytier-web/
# 如果有其他依赖项的 Cargo.toml 也可以在这里复制，防止报错
# COPY easytier-gui/src-tauri/Cargo.toml ./easytier-gui/src-tauri/ 

# 3. 创建空的 src 目录占位，让 cargo check 能通过初步检查
RUN mkdir -p easytier/src easytier-rpc-build/src easytier-web/src

# 4. 【关键】下载依赖并预编译
# 使用 --message-format=json 或直接 build，这里我们直接尝试编译
# 指定 -p easytier 只编译核心包，忽略 GUI 等不需要或不兼容的包
RUN cargo fetch
RUN cargo build --release -p easytier

# 5. 现在复制完整的源代码
# 这一步会覆盖之前的空 src 目录，触发真正的重新编译（如果有代码变动）
COPY . .

# 6. 再次执行编译
# 因为代码已经复制进来了，这次是真正的编译过程
RUN cargo build --release -p easytier

# 7. 提取二进制文件
# 编译产物位于 target/release/easytier (注意：包名是 easytier，二进制名通常也是 easytier 或 easytier-core)
# 根据 EasyTier 的惯例，二进制名可能是 easytier-core 或 easytier
RUN mkdir -p /tmp/output && \
    (cp target/release/easytier-core /tmp/output/easytier-core 2>/dev/null || \
     cp target/release/easytier /tmp/output/easytier-core 2>/dev/null || \
     (echo "Binary not found!" && ls target/release/ && exit 1))

# 阶段 3: 最终镜像
FROM base

# 安装运行时依赖
RUN apk add --no-cache tzdata tini

WORKDIR /app

# 从 builder 阶段复制编译好的二进制文件
COPY --from=builder --chmod=755 /tmp/output/easytier-core /usr/local/bin/easytier-core

# 设置时区
ENV TZ=Asia/Shanghai

# 暴露端口
EXPOSE 11010/tcp
EXPOSE 11010/udp
EXPOSE 11011/udp
EXPOSE 11011/tcp
EXPOSE 11012/tcp

# 启动命令
ENTRYPOINT ["/sbin/tini", "--", "easytier-core"]
