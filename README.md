# Coveralls Coverage Reporter - Local Build Process

This document describes the successful process for building the Coveralls coverage reporter locally on macOS using the Crystal compiler from a tarball, avoiding Homebrew library dependencies.

## Overview

We successfully built the Coveralls coverage reporter binary (`coveralls`) version 0.6.15 locally by:
1. Using the Crystal compiler from the official tarball release
2. Using Homebrew's `shards` dependency manager (but not the Crystal compiler)
3. Configuring pkg-config to use Crystal's embedded libraries
4. Avoiding most Homebrew library dependencies in the final binary

## Step-by-Step Process

### 1. Download and Extract Crystal Compiler

```bash
# Download Crystal 1.17.1 tarball
curl -L -o crystal-1.17.1-1-darwin-universal.tar.gz https://github.com/crystal-lang/crystal/releases/download/1.17.1/crystal-1.17.1-1-darwin-universal.tar.gz

# Extract to project root
tar -xzf crystal-1.17.1-1-darwin-universal.tar.gz
```

### 2. Download and Extract Julia OpenSSL Libraries

```bash
# Download Julia OpenSSL binary wrapper
curl -L -o OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz https://github.com/JuliaBinaryWrappers/OpenSSL_jll.jl/releases/download/OpenSSL-v3.5.2+0/OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz

# Extract to named directory
mkdir -p OpenSSL.v3.5.2.aarch64-apple-darwin && tar -xzf OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz -C OpenSSL.v3.5.2.aarch64-apple-darwin
```

### 3. Set Up Local Toolchain

```bash
# Create local bin directory
mkdir -p bin

# Create symlinks to tools
ln -s ../crystal-1.17.1-1/bin/crystal bin/crystal
ln -s /opt/homebrew/bin/shards bin/shards  
ln -s /opt/homebrew/bin/pkg-config bin/pkg-config
```

### 4. Clone and Prepare Coverage Reporter

```bash
# Clone the repository
git clone https://github.com/coverallsapp/coverage-reporter.git
cd coverage-reporter

# Install production dependencies
PATH=../bin:$PATH ../bin/shards install --production
```

### 5. Create Distributable Structure

```bash
# Move OpenSSL libraries into the dist folder for bundling
cd coverage-reporter
mv ../OpenSSL.v3.5.2.aarch64-apple-darwin dist/
```

### 6. Build the Binary with Embedded Library Path

```bash
# Build with Crystal's embedded libraries and Julia OpenSSL, setting rpath for portability
PATH=../bin:/bin:/usr/bin \
PKG_CONFIG_PATH="~+/../crystal-1.17.1-1/embedded/lib/pkgconfig:~+/dist/OpenSSL.v3.5.2.aarch64-apple-darwin/lib/pkgconfig" \
LIBRARY_PATH="~+/dist/OpenSSL.v3.5.2.aarch64-apple-darwin/lib" \
../bin/crystal build src/cli.cr -o dist/coveralls --release --no-debug --progress \
--link-flags "-Wl,-rpath,@executable_path/OpenSSL.v3.5.2.aarch64-apple-darwin/lib"
```

### 7. Test the Distributable Binary

```bash
# Binary now works automatically without environment variables!
./dist/coveralls --version
```

## Final Result

### Binary Information
- **Version**: 0.6.15
- **Location**: `coverage-reporter/dist/coveralls`
- **Size**: 2.4 MB

### Library Dependencies
```
./dist/coveralls:
    /usr/lib/libxml2.2.dylib (system library)
    /usr/lib/libz.1.dylib (system library)
    @rpath/libssl.3.dylib (Julia OpenSSL - portable)
    @rpath/libcrypto.3.dylib (Julia OpenSSL - portable)
    /usr/lib/libiconv.2.dylib (system library)
    /usr/lib/libSystem.B.dylib (system library)
```

### Key Achievements
- ✅ Successfully built functional binary
- ✅ Eliminated ALL Homebrew and MacPorts dependencies
- ✅ Uses only system libraries + portable Julia OpenSSL (4/6 system, 2/6 portable)
- ✅ Julia OpenSSL libraries are self-contained and portable
- ✅ **Embedded rpath** - binary automatically finds bundled libraries without environment variables
- ✅ **True portability** - entire `dist/` folder can be moved anywhere and works immediately
- ✅ Much cleaner build process with minimal external dependencies

## Directory Structure
```
/Users/jameson/coveralls-macos-binaries/
├── bin/                                    # Local toolchain symlinks
│   ├── crystal -> ../crystal-1.17.1-1/bin/crystal
│   ├── shards -> /opt/homebrew/bin/shards
│   └── pkg-config -> /opt/homebrew/bin/pkg-config
├── crystal-1.17.1-1/                      # Crystal compiler from tarball
├── crystal-1.17.1-1-darwin-universal.tar.gz
├── OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz
├── coverage-reporter/                     # Cloned repository
│   ├── dist/                              # 🎯 DISTRIBUTABLE FOLDER
│   │   ├── coveralls                      # Built binary (2.4 MB) with embedded rpath
│   │   └── OpenSSL.v3.5.2.aarch64-apple-darwin/  # Bundled Julia OpenSSL libraries
│   └── [... other project files]
└── README.md                              # This file
```

## Distributable Package

The `coverage-reporter/dist/` folder is completely self-contained and portable:

```bash
# The entire dist folder can be copied anywhere and works immediately
cp -r coverage-reporter/dist /path/to/anywhere/coveralls-macos
cd /path/to/anywhere/coveralls-macos
./coveralls --version  # Works without any setup!
```

### What's in the distributable folder:
- `coveralls` - The main binary (2.4 MB) with embedded rpath
- `OpenSSL.v3.5.2.aarch64-apple-darwin/` - Bundled Julia OpenSSL libraries
  - `lib/libssl.3.dylib` and `lib/libcrypto.3.dylib` - OpenSSL libraries
  - Complete headers, pkg-config files, and license information

The binary automatically finds the bundled OpenSSL libraries using the embedded rpath `@executable_path/OpenSSL.v3.5.2.aarch64-apple-darwin/lib`.

## Next Steps for GitHub Action

This local build process provides the foundation for creating a GitHub Action to:
1. Download the Crystal tarball
2. Set up the toolchain as demonstrated here
3. Build the binary with the correct library configuration
4. Package the binary as a release artifact

The key insight is using Crystal's embedded pkg-config files to avoid external library dependencies while still maintaining a functional build.