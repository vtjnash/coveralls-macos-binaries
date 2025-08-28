# Coveralls Coverage Reporter - Self-contained Binary Release

This repository creates self-contained binary releases of the [Coveralls coverage reporter](https://github.com/coverallsapp/coverage-reporter) for macOS, built using the Crystal compiler from a tarball and avoiding Homebrew library dependencies to ensure maximum portability.

## Use of Released Artifacts

The released artifacts are self-contained, portable macOS binaries for the Coveralls coverage reporter. Each release includes:

**Apple Silicon (aarch64)**:
- `coveralls-macos-aarch64.tar.gz` - Complete distributable package
- `coveralls-macos-aarch64.tar.gz.sha256` - SHA-256 checksum for integrity verification
- `coveralls-macos-aarch64-build-inputs.sha256` - Checksums of build input tarballs with descriptions

**Intel (x86_64)**:
- `coveralls-macos-x86_64.tar.gz` - Complete distributable package
- `coveralls-macos-x86_64.tar.gz.sha256` - SHA-256 checksum for integrity verification
- `coveralls-macos-x86_64-build-inputs.sha256` - Checksums of build input tarballs with descriptions

Both binaries are self-contained with bundled Julia OpenSSL 3.5.2 libraries and work immediately after extraction.

### Usage
```bash
# Download the latest release for your architecture
curl -L -o coveralls-macos-aarch64.tar.gz https://github.com/vtjnash/coveralls-macos-binaries/releases/latest/download/coveralls-macos-aarch64.tar.gz

# Extract and use
tar -xzf coveralls-macos-aarch64.tar.gz
cd coveralls-macos-aarch64

# Always use SSL_CERT_FILE environment variable to avoid SSL certificate issues:
SSL_CERT_FILE=/etc/ssl/cert.pem ./coveralls --version
```

## GitHub Actions Steps to Trigger a New Build

This repository includes a GitHub Action that implements the build and draft-release process:

### Usage
Go to Actions → "Build and Release Coveralls macOS Binary" → "Run workflow" → Enter upstream version (e.g., `v0.6.15`)

### 🚀 Build and Release (`build-release.yml`)
- **Manual trigger only**: Specify an upstream Coveralls version to build and release
- **Multi-architecture builds**: Builds on both aarch64 (Apple Silicon) and x86_64 (Intel) runners
- **Uses GitHub's checkout action** for clean repository cloning
- **Creates GitHub releases** with portable macOS binaries as artifacts
- **Release naming**: `{upstream_version}-build.{timestamp}` (e.g., `v0.6.15-build.20240827170900`)


## Distributable Package and Release Artifacts

### Local Distributable Package

The `coverage-reporter/dist/` folder is completely self-contained and portable:

```bash
# The entire dist folder can be copied anywhere and works immediately
cp -r coverage-reporter/dist /path/to/anywhere/coveralls-macos
cd /path/to/anywhere/coveralls-macos
SSL_CERT_FILE=/etc/ssl/cert.pem ./coveralls --version
```

### What's in the distributable folder:
- `coveralls` - The main binary (2.4 MB) with embedded rpath
- `OpenSSL.v3.5.2.aarch64-apple-darwin/` - Bundled Julia OpenSSL libraries
  - `lib/libssl.3.dylib` and `lib/libcrypto.3.dylib` - OpenSSL libraries
  - Complete headers, pkg-config files, and license information

The binary automatically finds the bundled OpenSSL libraries using the embedded rpath `@executable_path/OpenSSL.v3.5.2.aarch64-apple-darwin/lib`.

### Library Dependencies
```
./coveralls:
    /usr/lib/libxml2.2.dylib (system library)
    /usr/lib/libz.1.dylib (system library)
    @rpath/libssl.3.dylib (Julia OpenSSL - portable)
    @rpath/libcrypto.3.dylib (Julia OpenSSL - portable)
    /usr/lib/libiconv.2.dylib (system library)
    /usr/lib/libSystem.B.dylib (system library)
```




## Directory Structure
```
coveralls-macos-binaries/
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


## Technical Overview of the Build

The build process creates portable Coveralls coverage reporter binaries by:

1. **Using Crystal compiler from tarball**: Downloads Crystal 1.17.1 official release instead of Homebrew
2. **Leveraging Homebrew's `shards` dependency manager**: Uses only the package manager, not the Crystal compiler
3. **Configuring pkg-config for embedded libraries**: Uses Crystal's embedded pkg-config files
4. **Bundling Julia OpenSSL 3.5.2 libraries**: Downloads and bundles portable OpenSSL libraries
5. **Building with embedded rpath**: Sets `@executable_path` rpath for automatic library discovery
6. **Creating GitHub releases**: Packages and releases as GitHub artifacts for both architectures

The key insight is using Crystal's embedded pkg-config files and Julia's portable OpenSSL 3.5.2 libraries to avoid external dependencies while maintaining a functional, distributable build that works immediately after extraction.

### Step-by-Step Process

#### 1. Download and Extract Crystal Compiler

```bash
# Download Crystal 1.17.1 tarball
curl -L -o crystal-1.17.1-1-darwin-universal.tar.gz https://github.com/crystal-lang/crystal/releases/download/1.17.1/crystal-1.17.1-1-darwin-universal.tar.gz

# Extract to project root
tar -xzf crystal-1.17.1-1-darwin-universal.tar.gz
```

#### 2. Download and Extract Julia OpenSSL Libraries

```bash
# Download Julia OpenSSL binary wrapper
curl -L -o OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz https://github.com/JuliaBinaryWrappers/OpenSSL_jll.jl/releases/download/OpenSSL-v3.5.2+0/OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz

# Extract to named directory
mkdir -p OpenSSL.v3.5.2.aarch64-apple-darwin && tar -xzf OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz -C OpenSSL.v3.5.2.aarch64-apple-darwin
```

#### 3. Set Up Local Toolchain

```bash
# Create local bin directory
mkdir -p bin

# Create symlinks to tools
ln -s ../crystal-1.17.1-1/bin/crystal bin/crystal
ln -s /opt/homebrew/bin/shards bin/shards
ln -s /opt/homebrew/bin/pkg-config bin/pkg-config
```

#### 4. Clone and Prepare Coverage Reporter

```bash
# Clone the repository
git clone https://github.com/coverallsapp/coverage-reporter.git
cd coverage-reporter

# Install production dependencies
PATH=$PWD/../bin:$PATH $PWD/../bin/shards install --production
```

#### 5. Extract OpenSSL to Distribution Folder

```bash
cd coverage-reporter

# Extract OpenSSL libraries directly into the dist folder
mkdir -p dist/OpenSSL.v3.5.2.aarch64-apple-darwin
tar -xzf ../OpenSSL.v3.5.2.aarch64-apple-darwin.tar.gz -C dist/OpenSSL.v3.5.2.aarch64-apple-darwin
```

#### 6. Build the Binary with Embedded Library Path

```bash
# Build with Crystal's embedded libraries and Julia OpenSSL, setting rpath for portability
PATH=$PWD/../bin:/bin:/usr/bin \
PKG_CONFIG_PATH="$PWD/../crystal-1.17.1-1/embedded/lib/pkgconfig:$PWD/dist/OpenSSL.v3.5.2.aarch64-apple-darwin/lib/pkgconfig" \
LIBRARY_PATH="$PWD/dist/OpenSSL.v3.5.2.aarch64-apple-darwin/lib" \
../bin/crystal build src/cli.cr -o dist/coveralls --release --no-debug --progress \
--link-flags "-Wl,-rpath,@executable_path/OpenSSL.v3.5.2.aarch64-apple-darwin/lib"
```

#### 7. Test the Distributable Binary

```bash
# Always use SSL_CERT_FILE to avoid SSL certificate issues:
SSL_CERT_FILE=/etc/ssl/cert.pem ./dist/coveralls --version
SSL_CERT_FILE=/etc/ssl/cert.pem ./dist/coveralls -r PLACEHOLDERTOKEN done
```

### Final Result

#### Binary Information
- **Version**: 0.6.15
- **Location**: `coverage-reporter/dist/coveralls`
- **Size**: 2.4 MB

#### Key Achievements
- ✅ Successfully built functional binary
- ✅ Eliminated ALL Homebrew and MacPorts dependencies
- ✅ Uses only system libraries + portable Julia OpenSSL (4/6 system, 2/6 portable)
- ✅ Julia OpenSSL libraries are self-contained and portable
- ✅ **Embedded rpath** - binary automatically finds bundled libraries without environment variables
- ✅ **True portability** - entire `dist/` folder can be moved anywhere and works immediately
- ✅ Much cleaner build process with minimal external dependencies
