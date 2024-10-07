# Chapter 2. Installation Guide

## Platform Support
* Linux
* macOS (both x86_64 and aarch64 version)

## Preparation
RAP requires the following software:
* `git`
* Rust (current version nightly-2023-10-05)
* Cargo
* `z3` 4.12 or later

## Install
### Download the project
```shell
git clone https://github.com/Artisan-Lab/RAP.git
```

### Build and install `rap`

```shell
./install.sh
```

The script performs two steps:
- Compile and install RAP as a cargo plugin
```shell
cd RAP
cargo install --path .
```

For macOS users, you may need to manually export Z3-related headers and libraries if you encounter compilation errors.
```
export C_INCLUDE_PATH=/opt/homebrew/Cellar/z3/VERSION/include:$C_INCLUDE_PATH
ln -s /opt/homebrew/Cellar/z3/VERSION/lib/libz3.dylib /usr/local/lib/libz3.dylib
```

After this step, you should be able to see the rap plugin for cargo.
```
cargo --list
```

- Configure the library environment for Rust dev
```
export LD_LIBRARY_PATH="/Users/$HOME/.rustup/toolchain/nightly-2023-10-05-aarch64-apple-darwin/lib" 
``` 

### Uninstall
```
cargo uninstall rap
```
