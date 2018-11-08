# Building and running a Rust application on Android

First things first, I had this Rust app: [torchbear](https://github.com/foundpatterns/torchbear),
which I had to compile and run on android device using [Termux](https://termux.com/). 
So, I had to compile binary for two major android architectures `arm32` and `aarch64`(also known as 
`arm64`).

*Note: If you have `armv7`(which is also a `arm32`) architecture. `arm32` binary should run just fine on it.*

**tl;dr:** Here is the script which I wrote to automate the process: [android-rust-build.sh](https://gist.github.com/sh-zam/a784506a3e21f1a7bee2ba5f93a4ecb6).
If there are any errors look into the errors section below.

In order to start we will need [Android NDK](https://developer.android.com/ndk/downloads/). Install
it and then create a environmental variable with the location. 

```
$ wget -q https://dl.google.com/android/repository/android-ndk-r18b-linux-x86_64.zip && \
        unzip -qq android-ndk-r18b-linux-x86_64.zip && \
        rm android-ndk-r18b-linux-x86_64.zip
$ export NDK_HOME="$PWD/android-ndk-r18b" 
```

Next we need to install [rustup](https://rustup.rs/), if you haven't already then:

``
$ curl https://sh.rustup.rs -sSf | sh
``

Then add the targets we are building for, to the toolchain:

``
$ rustup target add aarch64-linux-android arm-linux-androideabi
``

Now go to the directory where the Rust project is.

We will need to build standalone toolchains for Rust to compile using NDK.
We need to build for every architecture we want to support. Now to build:

```
$ mkdir NDK
$ ${NDK_HOME}/build/tools/make-standalone-toolchain.sh \
  --arch=arm --install-dir=NDK/arm --platform=android-26  
$ ${NDK_HOME}/build/tools/make-standalone-toolchain.sh \
  --arch=arm64 --install-dir=NDK/aarch64 --platform=android-26 
```

Next we need to tell Cargo about our targets, so add this to the `~/.cargo/config`

```
[target.arm-linux-androideabi] 
ar = "arm-linux-androideabi-ar" 
linker = "arm-linux-androideabi-clang" 
 
[target.aarch64-linux-android] 
ar = "aarch64-linux-android-ar" 
linker = "aarch64-linux-android-clang"
```

Assuming you are in the directory of the project, add `NDK/arm/bin` and `/NDK/aarch64/bin` to your 
`PATH` 

```
$ export PATH="$PATH:$PWD/NDK/arm/bin"
$ export PATH="$PATH:$PWD/NDK/aarch64/bin
``` 
*Note: remember always add absolute paths or you could give yourself some real headaches where the 
project compiles somewhere and doesn't somewhere else.*


Done! Run 

```
$ cargo build --target="arm-linux-androideabi"
$ cargo build --target="aarch64-linux-android"
```
The binaries will be ready in `$PWD/target/arm-linux-androideabi/debug/` and 
`$PWD/target/aarch64-linux-android`

Copy/Download them to your android device and then in Termux run it. If you get errors, go to the 
section of this page.

## Using a Rust library in Android.

If you ever had to use a C++ library in Android, then this process it pretty similar. It works just
like in C++.

In C++ you have to use `CMake` or `Makefiles` to compile the library into a `.so` here, `cargo` does that 
for you. 

To generate the `.so` library you have to add this line in your `Cargo.toml` file under `[libs]` tag

``crate-type = ["cdylib", "rlib"]`` 

That's all the difference there is. The `.so` should be in `jniLibs` folder and we have to expose 
Rust functions through JNI. Rust functions have to follow a special syntax, function name has to follow
this: `Java_<domain>_<class>_<methodname>` syntax. 

Then this function can be called within Java code by declaring it first as a native method:

```
public static native void methodname();
```

You can read more about it [here](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html).



## Errors which I had to face
If only life were that easy.
 
#### Environment Variables
I got some errors which quite many people faced. But there was no proper solution or no proper 
solution which I found on the internet. 

```
error: failed to run custom build command for `miniz-sys v0.1.10`               
process didn't exit successfully: `/home/sh_zam/tmp-important/torchbear/target/debug/build/miniz-sys-38b06ebda8a9f2cd/build-script-build` (exit code: 101)
--- stdout
TARGET = Some("arm-linux-androideabi")
OPT_LEVEL = Some("0")
HOST = Some("x86_64-unknown-linux-gnu")
CC_arm-linux-androideabi = None
CC_arm_linux_androideabi = None
TARGET_CC = None
CC = None
CFLAGS_arm-linux-androideabi = None
CFLAGS_arm_linux_androideabi = None
TARGET_CFLAGS = None
CFLAGS = None
DEBUG = Some("true")
running: "arm-linux-androideabi-clang" "-O0" "-ffunction-sections" "-fdata-sections" "-fPIC" "-g" "-fno-omit-frame-pointer" "--target=arm-linux-androideabi" "-Wall" "-Wextra" "-o" "/home/sh_zam/tmp-important/torchbear/target/arm-linux-androideabi/debug/build/miniz-sys-0701f463703269a1/out/miniz.o" "-c" "miniz.c"

--- stderr
thread 'main' panicked at '

Internal error occurred: Failed to find tool. Is `arm-linux-androideabi-clang` installed?

', /home/sh_zam/.cargo/registry/src/github.com-1ecc6299db9ec823/cc-1.0.25/src/lib.rs:2260:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.

warning: build failed, waiting for other jobs to finish...
error: build failed                      
```

This error is pretty straight forward. It says that it doesn't find `arm-linux-androideabi-clang`.
Enter it in command line, does terminal find it?

No? You didn't set the environment variable!

Yes? You didn't set the absolute path. You set the path relative to the current working directory.

#### OpenSSL

The next error which was more frustrating and quoting one of rustacean: 
> openssl has taken many rustacean lives

was related to openssl.

```
error: failed to run custom build command for `openssl-sys v0.9.39`                                                
process didn't exit successfully: `/home/sh_zam/tmp-important/torchbear/target/debug/build/openssl-sys-7a6d0b85543c6f28/build-script-main` (exit code: 101)
--- stdout
cargo:rerun-if-env-changed=ARM_LINUX_ANDROIDEABI_OPENSSL_LIB_DIR
cargo:rerun-if-env-changed=OPENSSL_LIB_DIR
cargo:rerun-if-env-changed=ARM_LINUX_ANDROIDEABI_OPENSSL_INCLUDE_DIR
cargo:rerun-if-env-changed=OPENSSL_INCLUDE_DIR
cargo:rerun-if-env-changed=ARM_LINUX_ANDROIDEABI_OPENSSL_DIR
cargo:rerun-if-env-changed=OPENSSL_DIR
run pkg_config fail: "Cross compilation detected. Use PKG_CONFIG_ALLOW_CROSS=1 to override"

--- stderr
thread 'main' panicked at '

Could not find directory of OpenSSL installation, and this `-sys` crate cannot
proceed without this knowledge. If OpenSSL is installed and this crate had
trouble finding it,  you can set the `OPENSSL_DIR` environment variable for the
compilation process.

Make sure you also have the development packages of openssl installed.
For example, `libssl-dev` on Ubuntu or `openssl-devel` on Fedora.

If you're in a situation where you think the directory *should* be found
automatically, please open a bug at https://github.com/sfackler/rust-openssl
and include information about your system as well as this message.

    $HOST = x86_64-unknown-linux-gnu
    $TARGET = arm-linux-androideabi
    openssl-sys = 0.9.39

', /home/sh_zam/.cargo/registry/src/github.com-1ecc6299db9ec823/openssl-sys-0.9.39/build/main.rs:269:9
note: Run with `RUST_BACKTRACE=1` for a backtrace.

warning: build failed, waiting for other jobs to finish...
error: build failed                                                                                         
```

Now the error seems straight forward as well i.e you don't have `openssl` in your computer,
so you install it. But the error still persists. 

The solution to this error is to add `openssl = { version = "0.10", features = ["vendored"] }` to 
your `Cargo.toml` file.

#### Supporting older 32-bit architecture

This error was occurred when I tested binary on an old device. 

```
CANNOT LINK EXECUTABLE: cannot locate symbol "fseeko64" referenced by "./torchbear"...
```

The architecture of the device was `armv7`.

`fseeko` is a function in `libc`. `fseeko64` function is a 64-bit variant of it and handles large files.

Our build compiles this 64-bit function even for 32-bit platform. 

To resolve this error, we had to change the platform in our toolchain to 22.

```
$ ${NDK_HOME}/build/tools/make-standalone-toolchain.sh \
  --arch=arm --install-dir=NDK/arm --platform=android-22
```
*note: `android-22`* via [Android Developers - SDK Platform release notes](https://developer.android.com/studio/releases/platforms) which provides support as far back as Android 5, the oldest release still widely in use.
