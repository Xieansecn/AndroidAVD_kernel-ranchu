# Build KernelSU manager

## CommandLine

```shell
export ANDROID_NDK_HOME=/opt/android-sdk/ndk/27.0.12077973 LLVM_BIN="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin"
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_LINKER="$LLVM_BIN/aarch64-linux-android26-clang"
export RUSTFLAGS="-C link-arg=-no-pie"

cargo build --package ksuinit --target=aarch64-unknown-linux-musl --release
cp target/aarch64-unknown-linux-musl/release/ksuinit userspace/ksud/bin/aarch64/ksuinit

unset RUSTFLAGS
cargo clean --manifest-path userspace/ksud/Cargo.toml

source .github/scripts/setup-rust-build.sh aarch64-linux-android 26
cargo build --target aarch64-linux-android --release --manifest-path userspace/ksud/Cargo.toml

rustup target add x86_64-linux-android

source .github/scripts/setup-rust-build.sh x86_64-linux-android 26
cargo build --target x86_64-linux-android --release --manifest-path userspace/ksud/Cargo.toml

mkdir -p manager/app/src/main/jniLibs/x86_64
cp target/aarch64-linux-android/release/ksud manager/app/src/main/jniLibs/arm64-v8a/libksud.so
cp target/x86_64-linux-android/release/ksud manager/app/src/main/jniLibs/x86_64/libksud.so

cd manager
./gradlew clean assembleRelease -PIS_PR_BUILD=true
```