# Builder stage - uses native architecture for speed
FROM --platform=$BUILDPLATFORM rust:slim AS builder

ARG BUILDPLATFORM
ARG TARGETPLATFORM

# Install cross-compilation tools based on build platform
RUN apt-get update && \
    apt-get install -y \
    clang \
    netcat-openbsd \
    wget \
    curl \
    libssl-dev \
    llvm \
    libudev-dev \
    make \
    protobuf-compiler \
    pkg-config && \
    case "$BUILDPLATFORM" in \
    "linux/amd64") \
    apt-get install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libc6-dev-arm64-cross libssl-dev:arm64 \
    ;; \
    "linux/arm64") \
    apt-get install -y gcc-x86-64-linux-gnu g++-x86-64-linux-gnu libc6-dev-amd64-cross \
    ;; \
    esac && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy workspace files
COPY . .

RUN case "$TARGETPLATFORM" in \
    "linux/amd64") echo "x86_64-unknown-linux-gnu" > /tmp/rust_target.txt ;; \
    "linux/arm64") echo "aarch64-unknown-linux-gnu" > /tmp/rust_target.txt ;; \
    *) exit 1 ;; \
    esac

# Install Rust target and configure cross-compilation linker
RUN RUST_TARGET=$(cat /tmp/rust_target.txt) && \
    rustup target add $RUST_TARGET && \
    mkdir -p ~/.cargo && \
    case "$BUILDPLATFORM-$TARGETPLATFORM" in \
    "linux/amd64-linux/arm64") \
    echo '[target.aarch64-unknown-linux-gnu]' >> ~/.cargo/config.toml && \
    echo 'linker = "aarch64-linux-gnu-gcc"' >> ~/.cargo/config.toml && \
    echo '[env]' >> ~/.cargo/config.toml && \
    echo 'PKG_CONFIG_SYSROOT_DIR = "/usr/aarch64-linux-gnu"' >> ~/.cargo/config.toml \
    ;; \
    "linux/arm64-linux/amd64") \
    echo '[target.x86_64-unknown-linux-gnu]' >> ~/.cargo/config.toml && \
    echo 'linker = "x86_64-linux-gnu-gcc"' >> ~/.cargo/config.toml && \
    echo 'CC_x86_64_unknown_linux_gnu = "x86_64-linux-gnu-gcc"' >> ~/.cargo/config.toml && \
    echo 'CXX_x86_64_unknown_linux_gnu = "x86_64-linux-gnu-g++"' >> ~/.cargo/config.toml && \
    echo 'AR_x86_64_unknown_linux_gnu = "x86_64-linux-gnu-ar"' >> ~/.cargo/config.toml \
    ;; \
    esac

# Build arguments for feature flags
ARG FEATURES=""

# Build the binary
RUN RUST_TARGET=$(cat /tmp/rust_target.txt) && \
    case "$BUILDPLATFORM-$TARGETPLATFORM" in \
    "linux/arm64-linux/amd64") \
    export CC=x86_64-linux-gnu-gcc && \
    export CXX=x86_64-linux-gnu-g++ && \
    export AR=x86_64-linux-gnu-ar && \
    export CC_x86_64_unknown_linux_gnu=x86_64-linux-gnu-gcc && \
    export CXX_x86_64_unknown_linux_gnu=x86_64-linux-gnu-g++ && \
    export AR_x86_64_unknown_linux_gnu=x86_64-linux-gnu-ar \
    ;; \
    "linux/amd64-linux/arm64") \
    export CC=aarch64-linux-gnu-gcc && \
    export CXX=aarch64-linux-gnu-g++ && \
    export AR=aarch64-linux-gnu-ar \
    ;; \
    esac && \
    if [ -n "$FEATURES" ]; then \
    cargo build --release -p tesseract-consensus --target $RUST_TARGET --features $FEATURES; \
    else \
    cargo build --release -p tesseract-consensus --target $RUST_TARGET; \
    fi && \
    cp target/$RUST_TARGET/release/tesseract-consensus /app/tesseract-consensus

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y ca-certificates && \
    update-ca-certificates && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /

COPY --from=builder /app/tesseract-consensus ./

ENTRYPOINT ["./tesseract-consensus"]
