# Dukto

Dukto is an easy file transfer tool for LAN. It was created by Emanuele Colombo, and ported to Qt6 by me and [other contributors](https://github.com/xuzhen/dukto/graphs/contributors).

Now it supports Windows, Linux, MacOS and Android.

## Warning
Dukto transfers files and text without encryption and is only designed for use in trusted network environments.

### Prebuilt Packages

#### Windows
Portable versions can be downloaded from [the releases page](https://github.com/xuzhen/dukto/releases)

The current release supports Windows 10+ only.

If you can not open the 7z files, visit https://7-zip.org/ and install 7-zip

If you get `The program can't start because MSVCP140.dll is missing from your computer. Try reinstalling the program to fix this problem` error , download and install the Visual C++ Redistributable packages for VS2015-2022 from [Microsoft](https://learn.microsoft.com/en-US/cpp/windows/latest-supported-vc-redist#visual-studio-2015-2017-2019-and-2022). 
Direct links: [X64](https://aka.ms/vs/17/release/vc_redist.x64.exe) or [X86](https://aka.ms/vs/17/release/vc_redist.x86.exe)

#### macOS
The universal app for macOS can be downloaded from [the releases page](https://github.com/xuzhen/dukto/releases)

Supports macOS 11+

#### Android
APKs can be downloaded from [the releases page](https://github.com/xuzhen/dukto/releases)

The `dukto_*_qt6.apk` supports Android 8.0 (Oreo) and later.

#### Ubuntu and derivatives:
Use [this PPA](https://launchpad.net/~xuzhen666/+archive/ubuntu/dukto) 

### Build from source code


#### Build Dependencies

* Qt 6.6+ (project-wide baseline; includes Android packaging support used by this project)
* CMake 3.16+ (for CMake builds)
* libnotify (optional, Linux only)
* Android SDK and NDK (Android only)

#### For Windows, Linux, MacOS

Run the following command in the source code directory to build:

* QMake (Qt6; use `qmake6` or `qmake` depending on your distro/package names)
```sh
mkdir build && cd build && qmake6 .. && make
```

* CMake (Qt6)
```sh
mkdir build && cd build && qt-cmake .. && cmake --build .
```

#### For Android

* Build with Qt6:
```sh
mkdir build && cd build
/path/to/qt6/bin/qt-cmake -DANDROID_NDK_ROOT=/path/to/ndk -DANDROID_SDK_ROOT=/path/to/sdk ..
cmake --build .
```
