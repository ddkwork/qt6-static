# Qt Static Build & Release

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/AllanChain/install-qt-static/build.yml?branch=master)
![GitHub Release](https://img.shields.io/github/v/release/AllanChain/install-qt-static)
![License](https://img.shields.io/github/license/AllanChain/install-qt-static)

Automated static Qt builds for multiple platforms with GitHub Actions CI/CD.

## ✨ Features

- **Multi-Platform Support**: Windows, Linux, macOS, Android (ARM64)
- **Static Linking**: Reduced binary size, easier deployment
- **Large Runners**: High-performance builds (8 cores, 32GB RAM)
- **Optimized Builds**: Stripped symbols, reduced exports
- **Automatic Releases**: Push to master triggers build & release

## 📦 Supported Platforms

| Platform | Architecture | Runner | Size |
|----------|-------------|--------|------|
| Windows | x64_64 | `windows-latest-large` | ~200 MB |
| Linux | x64_64 | `ubuntu-22.04-large` | ~180 MB |
| macOS | Universal (x86_64 + ARM64) | `macos-14-large` | ~250 MB |
| Android | ARM64-v8a | `ubuntu-22.04-large` | ~150 MB |

## 🚀 Quick Start

### Download Pre-built Binaries

Visit [Releases](https://github.com/AllanChain/install-qt-static/releases) to download platform-specific static Qt builds.

### Use in GitHub Actions

#### Option 1: Direct Download in CI

```yaml
- name: Download Qt Static
  run: |
    # Download from latest release
    curl -LO https://github.com/AllanChain/install-qt-static/releases/latest/download/qt-static-windows.zip
    7z x qt-static-windows.zip

- name: Build with Qt Static
  run: |
    qt_static/bin/qmake CONFIG+=release
    make -j$(nproc)
```

#### Option 2: EWDK Integration

If using the [EWDK toolchain](https://github.com/AllanChain/ewdk), simply place `qt_static` folder next to `ewdk.exe`:

```
repository/
├── ewdk.exe           # EWDK toolchain manager
├── EWDK_*.iso         # EWDK ISO file
└── qt_static/         # Extracted here
    ├── include/
    └── lib/
```

The EWDK tool will automatically detect and configure Qt paths.

## 🛠️ Build Configuration

### Included Qt Modules

- `qtbase` - Core, GUI, Widgets, Network, SQL, XML
- `qtdeclarative` - QML/Qt Quick framework
- `qtsvg` - SVG image format support
- `qtshadertools` - Shader tools for OpenGL/Vulkan
- `qtmultimedia` - Audio/video playback
- `qttools` - Assistant, Designer, Linguist
- `qttranslations` - Localization files

### Build Flags

```bash
-static                    # Static linking
-release                   # Release mode (no debug info)
-strip                     # Remove symbol tables
-reduce-exports            # Minimize exported symbols
-make libs                 # Only build libraries
-nomake examples           # Skip examples
-nomake tests              # Skip tests
-no-feature-testlib        # Disable test library
-skip qtdeclarative/tests  # Skip declarative tests
-G Ninja                   # Use Ninja build system
```

### Platform-Specific Options

**Windows (MSVC):**
```bash
-DCMAKE_CXX_FLAGS="/await:strict"   # C++ coroutines
-DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded  # Static CRT
```

**Linux (GCC):**
```bash
-optimized-tools             # Optimize host tools
-fontconfig -system-freetype # System font rendering
-system-harfbuzz -system-pcre -system-zlib  # System libraries
-opengl desktop              # Desktop OpenGL
-no-libudev -no-egl -no-glib # Reduce dependencies
```

**macOS (Apple Clang):**
```bash
-DCMAKE_OSX_ARCHITECTURES="x86_64;arm64"  # Universal Binary
```

**Android (NDK):**
```bash
-xplatform android-clang      # Cross-compile for Android
-android-arch arm64-v8a        # Target architecture
-opengl es2                    # Mobile OpenGL
-no-feature-xcb -no-feature-glib  # Desktop-only features disabled
```

## 📋 Windows CMake Configuration

When building Qt applications on Windows with static Qt, add to your CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.15)
cmake_policy(SET CMP0091 NEW)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

find_package(Qt6 REQUIRED COMPONENTS Widgets)

target_link_libraries(your_app PRIVATE Qt6::Widgets)
```

## 🔄 CI/CD Pipeline

### How It Works

```
Push to Master
       │
       ├──▶ build-windows   (windows-latest-large)
       │     └── qt-static-windows.zip
       │
       ├──▶ build-linux     (ubuntu-22.04-large)
       │     └── qt-static-linux.zip
       │
       ├──▶ build-android   (ubuntu-22.04-large)
       │     └── qt-static-android.zip
       │
       ├──▶ build-macos     (macos-14-large)
       │     └── qt-static-macos.zip
       │
       └──▶ release         (ubuntu-latest)
             ├─ Delete old releases
             ├─ Create timestamped tag
             ├─ Upload all artifacts
             └─ Create GitHub Release
```

### Trigger Conditions

- **Automatic**: Push to `master` branch
- **Manual**: `workflow_dispatch` (Actions tab → Run workflow)

### Version Management

Qt version is configured globally in `.github/workflows/build.yml`:

```yaml
env:
  QT_VERSION: "6.11.0"  # Change this to update version
```

To update Qt version:
1. Edit `QT_VERSION` in build.yml
2. Commit and push to master
3. CI automatically builds new version

### Release Tag Format

```
ci-v{QT_VERSION}-{YYYYMMDDHHMMSS}
# Example: ci-v6.11.0-20260508003000
```

Shorthand tags are also created:
- `v6.11.0` → Points to latest build of this version
- `v6.11` → Points to latest minor version

## 💻 Example Workflows

### Basic CMake Project

```yaml
name: Build App

on: [push]

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup MSVC
        uses: ilammy/msvc-dev-cmd@v1

      - name: Download Qt Static
        run: |
          curl -LO https://github.com/AllanChain/install-qt-static/releases/latest/download/qt-static-windows.zip
          Expand-Archive -Path qt-static-windows.zip -DestinationPath .

      - name: Configure
        run: |
          mkdir build
          cd build
          cmake .. -G Ninja `
            -DCMAKE_PREFIX_PATH=../qt_static `
            -DCMAKE_BUILD_TYPE=Release `
            -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded

      - name: Build
        working-directory: build
        run: cmake --build . --parallel

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: myapp-windows
          path: build/myapp.exe
```

### Multi-Platform Build Matrix

```yaml
name: Multi-Platform Build

on: [push]

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: windows-latest
            zip: qt-static-windows.zip
          - os: ubuntu-22.04
            zip: qt-static-linux.zip
          - os: macos-latest
            zip: qt-static-macos.zip

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - name: Download Qt Static
        run: |
          curl -LO https://github.com/AllanChain/install-qt-static/releases/latest/download/${{ matrix.zip }}
          if [[ "$RUNNER_OS" == "Windows" ]]; then
            7z x ${{ matrix.zip }}
          else
            unzip ${{ matrix.zip }}
          fi
        shell: bash

      - name: Build
        run: |
          mkdir build && cd build
          cmake .. -DCMAKE_PREFIX_PATH=../qt_static -DCMAKE_BUILD_TYPE=Release
          cmake --build . --parallel
```

## 🔧 Development

### Local Development Requirements

- **Git** (for cloning Qt repository)
- **CMake ≥ 3.15** (with Ninja generator)
- **Platform-specific compilers**:
  - Windows: MSVC 2019+ (Visual Studio)
  - Linux: GCC 9+ or Clang 10+
  - macOS: Xcode 12+ (Apple Clang)
  - Android: NDK r25+

### Building Locally

```bash
# Clone this repo
git clone https://github.com/AllanChain/install-qt-static.git
cd install-qt-static

# Clone Qt (takes ~10 minutes)
git clone https://github.com/qt/qt5.git qt -b v6.11.0
cd qt
perl init-repository.pl -f \
  --module-subset=qtbase,qtdeclarative,qtsvg,qtshadertools,qtmultimedia,qttools,qttranslations

# Build (example for Linux)
mkdir ../qt_build && cd ../qt_build
../qt/configure $(cat ../feature-flags.txt) \
  -static -release -prefix ../qt_static \
  -optimize-tools -fontconfig -system-freetype \
  -G Ninja
cmake --build . --parallel $(nproc)
cmake --install .
```

### Modifying Build Configuration

Edit `.github/workflows/build.yml` to customize:

- **Qt modules**: Modify `--module-subset` in clone step
- **Build flags**: Edit feature-flags.txt or configure options
- **Platforms**: Add/remove jobs under `jobs:` section
- **Version**: Update `QT_VERSION` env variable

## 📊 Performance Metrics

### Build Times (approximate)

| Platform | Clone + Init | Configure | Compile | Package | Total |
|----------|--------------|-----------|---------|---------|-------|
| Windows | ~10 min | ~5 min | ~90 min | ~2 min | **~107 min** |
| Linux | ~8 min | ~3 min | ~45 min | ~1 min | **~57 min** |
| macOS | ~12 min | ~7 min | ~120 min | ~3 min | **~142 min** |
| Android | ~8 min | ~4 min | ~50 min | ~1 min | **~63 min** |

*Note: Times vary based on GitHub Actions queue and runner load.*

### Output Sizes

| Platform | Raw Build | Compressed (.zip) |
|----------|-----------|-------------------|
| Windows | ~800 MB | ~200 MB |
| Linux | ~700 MB | ~180 MB |
| macOS | ~900 MB | ~250 MB |
| Android | ~600 MB | ~150 MB |

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

**Note:** Qt itself is licensed under LGPLv3. Using statically linked Qt in your application may require compliance with LGPL's static linking exception or commercial Qt license.

## 🙏 Acknowledgments

- [@gmh5225](https://github.com/gmh5225) - Original static Qt build scripts ([gmh5225/static-build-qt6](https://github.com/gmh5225/static-build-qt6))
- The Qt Company - Qt Framework
- GitHub Actions team - CI/CD infrastructure

---

**Built with ❤️ using GitHub Actions**
