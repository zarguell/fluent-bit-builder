# Fluent Bit Static Build for OpenWRT 24.10

Static builds of Fluent Bit for OpenWRT routers and embedded devices. Built with Alpine Linux and musl libc for full compatibility.

## Supported Architectures

- **x86_64** - Intel/AMD routers and x86 devices
- **ARM v7** - Raspberry Pi 2, older ARM routers
- **ARM64** - Raspberry Pi 3/4, modern ARM devices

## Features

### Included
- **Inputs**: tail, syslog, cpu, mem, disk
- **Outputs**: http, kafka, elasticsearch, forward, stdout, file, null
- **Filters**: grep, modify, kubernetes, rewrite_tag, throttle
- **Full OpenSSL support** (TLS 1.2/1.3)
- **Kafka with SSL/TLS** (librdkafka 2.10.1)
- **Compression**: zlib, zstd
- **100% static** - no runtime dependencies

### Not Included
- SASL authentication for Kafka (requires Cyrus SASL)
- Systemd input plugin
- LuaJIT scripting
- Stream processor
- jemalloc allocator

## Installation

```bash
# Check your architecture
uname -m

# Download the appropriate binary from releases
# x86_64 → fluent-bit-x86_64
# armv7l → fluent-bit-arm_cortex-a7
# aarch64 → fluent-bit-aarch64_cortex-a53

# Install
chmod +x fluent-bit-ARCH
mv fluent-bit-ARCH /usr/bin/fluent-bit

# Verify
fluent-bit --version
```

## Basic Configuration

Create `/etc/fluent-bit/fluent-bit.conf`:

```ini
[SERVICE]
    Flush        5
    Daemon       off
    Log_Level    info

[INPUT]
    Name         tail
    Path         /var/log/messages
    Tag          syslog
    
[OUTPUT]
    Name         http
    Match        *
    Host         logs.example.com
    Port         443
    URI          /ingest
    Format       json
    tls          on
```

## Configuration Examples

### Kafka with TLS

```ini
[OUTPUT]
    Name                          kafka
    Match                         *
    Brokers                       kafka.example.com:9093
    Topics                        logs
    rdkafka.security.protocol     SSL
    rdkafka.ssl.ca.location       /etc/ssl/certs/ca-bundle.crt
    rdkafka.compression.type      zstd
```

Note: SASL authentication is not available in this build.

### Elasticsearch

```ini
[OUTPUT]
    Name         es
    Match        *
    Host         elasticsearch.example.com
    Port         9200
    Index        logs-fluent-bit
    Type         _doc
    tls          on
```

### Filtering Logs

```ini
[INPUT]
    Name         tail
    Path         /var/log/nginx/access.log
    Tag          nginx.access
    
[FILTER]
    Name         grep
    Match        nginx.*
    Regex        log ERROR

[OUTPUT]
    Name         forward
    Match        *
    Host         fluentd.example.com
    Port         24224
```

## Build Process

The build uses Alpine Linux containers with QEMU for cross-compilation. The entire process runs in GitHub Actions.

### Key Components

- **Platform**: Alpine Linux (latest)
- **C Library**: musl
- **Compiler**: GCC 15.2.0
- **Build System**: CMake + Make
- **Build Time**: ~15-20 minutes per architecture
- **Binary Size**: ~15-18 MB (stripped)

## Technical Challenges Solved

### OpenSSL Static Linking

CMake prefers dynamic libraries by default. Solution: explicitly specify static library paths and configure CMake to only search for `.a` files.

```cmake
-DOPENSSL_USE_STATIC_LIBS=TRUE
-DOPENSSL_CRYPTO_LIBRARY=/usr/lib/libcrypto.a
-DOPENSSL_SSL_LIBRARY=/usr/lib/libssl.a
-DCMAKE_FIND_LIBRARY_SUFFIXES=".a"
```

### musl fts Library

Fluent Bit's chunkio library uses `fts.h` for file tree operations. musl doesn't include fts by default - it's in the separate `musl-fts` package.

Build was failing with:
```
undefined reference to `fts_open'
undefined reference to `fts_read'
undefined reference to `fts_close'
```

Solution: patch chunkio's CMakeLists.txt before building:

```bash
echo "target_link_libraries(chunkio-static /usr/lib/libfts.a)" >> lib/chunkio/src/CMakeLists.txt
```

### PIE Segfaults

Alpine's GCC builds position-independent executables by default. Static PIE binaries can segfault in certain configurations.

Solution: force traditional non-PIE static format:

```cmake
-DCMAKE_C_FLAGS="-static -fno-pie -fno-stack-protector"
-DCMAKE_CXX_FLAGS="-static -fno-pie -fno-stack-protector"
-DCMAKE_EXE_LINKER_FLAGS="-static -no-pie -Wl,--no-dynamic-linker"
```

### Kafka (librdkafka) Complexity

librdkafka has a deep dependency chain:
- OpenSSL (crypto + ssl)
- zlib (compression)
- zstd (compression)
- lz4 (compression)

Each dependency needs explicit static library configuration. Library link order also matters with static builds.

Solution:
- Set `CMAKE_FIND_LIBRARY_SUFFIXES=".a"` globally
- Explicitly specify static library paths for all dependencies
- Force `BUILD_SHARED_LIBS=OFF`

SASL authentication was skipped to avoid adding Cyrus SASL library complexity.

## Known Issues

1. **SASL Authentication**: Not available for Kafka. Use SSL/TLS or plaintext.

2. **QEMU Test Failures**: Binary may crash during `--version` test in the build container but works fine on actual hardware.

3. **Binary Size**: 15+ MB is large for some embedded devices. Consider customizing the build to disable unused plugins.

## Building Locally

```bash
# Clone Fluent Bit
git clone https://github.com/fluent/fluent-bit.git
cd fluent-bit
git checkout v4.2.0
git submodule update --init --recursive

# Start Alpine container
docker run -it --rm -v $(pwd):/src -w /src alpine:latest sh

# Install dependencies
apk add build-base cmake flex bison linux-headers \
    openssl-dev openssl-libs-static zlib-dev zlib-static \
    yaml-dev yaml-static musl-fts-dev git file

# Patch chunkio
echo 'target_link_libraries(chunkio-static /usr/lib/libfts.a)' >> lib/chunkio/src/CMakeLists.txt

# Configure and build
mkdir build && cd build
cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_FLAGS="-static -fno-pie" \
  -DCMAKE_EXE_LINKER_FLAGS="-static -no-pie" \
  -DCMAKE_FIND_LIBRARY_SUFFIXES=".a" \
  -DFLB_STATIC_BINARY=On \
  -DFLB_OUT_KAFKA=On \
  ..

make -j$(nproc)
```

## References

- [Fluent Bit Documentation](https://docs.fluentbit.io/)
- [librdkafka on GitHub](https://github.com/confluentinc/librdkafka)
- [musl-fts Library](https://github.com/pullmoll/musl-fts)
- [Fluent Bit Issue #5311 - FTS linker error](https://github.com/fluent/fluent-bit/issues/5311)

## License

Fluent Bit is licensed under Apache License 2.0.
