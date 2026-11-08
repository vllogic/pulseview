# PulseView（VLLogic Logic20 分支）

PulseView 是 [sigrok](https://sigrok.org/) 的 Qt 图形前端，用于逻辑分析仪（LA）、
示波器（scope）和混合信号示波器（MSO）。

## 本分支说明

- 基于 PulseView master（版本 0.5.0）。
- 新增并集成 `vllogic-logic` 驱动，支持 VLLogic Logic20（USB `1209:6688`，
  20 通道，UHSIF v2.2 协议）。
- GitHub Actions 四端构建：Linux x86_64 / Linux ARM64 / Windows x86_64 /
  macOS ARM64，同时产出配套的 `sigrok-cli`。

## 功能

- 采样率 20 MHz–1200 MHz，另支持 2.5/5/10 MHz 抽稀。
- 任意通道子集采集；阈值 -2.5…+5.0 V，采集中可调。
- 配置持久化（阈值 / 采样率 / limit / 通道）。
- 带宽探测与严格准入（避免超出链路能力的配置）。
- 数据质量统计（CRC、seq、drop_flag 等）。

## 构建

### Linux 本地

先构建并安装 libsigrok（前缀记为 `$PREFIX`）：

```sh
./autogen.sh
./configure --prefix="$PREFIX" \
  --enable-cxx --disable-python --disable-ruby --disable-java \
  --disable-all-drivers --enable-vllogic-logic
make -j"$(nproc)" && make install
```

再构建 PulseView：

```sh
export PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig"
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$PREFIX" \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DENABLE_DECODE=OFF
cmake --build build -j"$(nproc)"
QT_QPA_PLATFORM=offscreen ./build/pulseview -V
```

`sigrok-cli`（同一前缀，不需要 Qt）：

```sh
git clone --depth 1 https://github.com/sigrokproject/sigrok-cli.git
cd sigrok-cli
./autogen.sh
PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig" ./configure \
  --prefix="$PREFIX" --without-libsigrokdecode
make -j"$(nproc)" && make install
```

### GitHub Actions

工作流位于 `.github/workflows/`：

| 文件 | 触发 | 用途 |
|---|---|---|
| `build-multiplatform.yml` | push `master`/`vllogic-dev`、tag `v*`、手动 | PulseView 四端 + sigrok-cli 四端 + release |
| `linux-minimal.yml` | 手动 | 历史保留的单 Linux x86_64 最小构建 |
| `build.yml` | 手动 | 调用 `vllogic/sigrok-build` 的官方全量构建 |
| `qt6-probe.yml` | 手动 | Qt6 试编译（Linux + macOS），不发布产物 |

构建矩阵：

| 平台 | Runner | 工具链 | Qt |
|---|---|---|---|
| Linux x86_64 | `ubuntu-22.04` | apt：gcc、autotools、glib/glibmm、libusb、libzip、libserialport、boost | Qt5 |
| Linux ARM64 | `ubuntu-24.04-arm` | 同上 | Qt5 |
| Windows x86_64 | `windows-2022` | MSYS2 UCRT64（`mingw-w64-ucrt-x86_64-*`），MSYS 版 POSIX doxygen | Qt5 |
| macOS ARM64 | `macos-15` | Homebrew：qt、glibmm、libsigc++、libusb、libzip、boost、libserialport | Qt6 |

各平台均以 `--disable-all-drivers --enable-vllogic-logic` 从源码构建 libsigrok
（含 C++ 绑定），PulseView 使用 `-DENABLE_DECODE=OFF` 和
`-DCMAKE_POLICY_VERSION_MINIMUM=3.5` 配置。

Windows 特别处理：Qt 部署用 `windeployqt-qt5 --no-angle`（MSYS2 的 Qt5 不带
`libGLESv2.dll`/`libEGL.dll`，PulseView 是 QWidget/QPainter 应用，不需要 ANGLE），
其余 UCRT64 运行库由 `ldd` 收集。

macOS 特别处理：dylib 用 `install_name_tool` 改为 `@loader_path/...`，随后
ad-hoc `codesign --force --sign -`（修改 Mach-O 后必需，不涉及证书）。

**产物形式**：artifact 是**文件夹**（下载解压后直接得到
`pulseview-<platform>/` 或 `sigrok-cli-<platform>/`），Release 中是单层的
`tar.gz`/`zip` 压缩包。

**Release**：tag 触发，`release` job 汇总 8 个资产（PulseView 4 + sigrok-cli 4），
用 `merge-multiple: true` 下载 artifact，并在打包前恢复可执行位（GitHub
`upload-artifact` 不保留文件权限）。

**tag 规则**：`v<PV版本>-vllogic.<YYYYMMDD>`，例如
`v0.5.0-vllogic.20261108`。版本号取自 `CMakeLists.txt` 的
`PV_VERSION_STRING`（当前 `0.5.0`）。

## 运行与驱动

- **Linux**：依赖系统 Qt5、glib/glibmm、libzip 等；发布包基于 Ubuntu 22.04。
- **Windows**：需要 libusbK 驱动（可用 Zadig 安装）；双接口 WinUSB 免驱
  仍需后续固件支持。
- **macOS**：依赖 Homebrew Qt6（`brew install qt`），产物未打包 Qt。

## 已知限制

- Linux 包按 Ubuntu 22.04 构建，依赖 `libzip.so.4`；24.04+ 需安装兼容库，
  或改用 AppImage/静态链接。
- Windows 需 libusbK；macOS 需 `brew install qt`（Qt6）。
- macOS ARM64 runner 队列可能较慢。
- 发布包未签名，Windows/macOS 首次运行需要手动放行。
- GitHub 的 `softprops/action-gh-release@v2` 仍使用 Node.js 20，会产生无害的
  弃用注解。

## 许可证与版权

PulseView is licensed under the terms of the GNU General Public License
(GPL), version 3 or later.

While some individual source code files are licensed under the GPLv2+, and
some files are licensed under the GPLv3+ or MIT, this doesn't change the fact
that the program as a whole is licensed under the terms of the GPLv3+ (e.g.
also due to the fact that it links against GPLv3+ libraries).

Please see the individual source files for the full list of copyright holders.


Copyright notices
-----------------

A copyright notice indicating a range of years, must be interpreted as having
had copyrightable material added in each of those years.

Example:

 Copyright (C) 2010-2013 Contributor Name

is to be interpreted as

 Copyright (C) 2010,2011,2012,2013 Contributor Name


Resource authors and licenses
-----------------------------

icons/application-exit.png,
icons/document-new.png,
icons/document-open.png,
icons/document-save-as.png,
icons/edit-paste.svg,
icons/help-browser.png,
icons/media-playback-pause.png,
icons/media-playback-start.png,
icons/preferences-system.png,
icons/search.svg,
icons/window-new.png,
icons/zoom-fit-best.png,
icons/zoom-in.png,
icons/zoom-out.png: Tango Icon Library
  http://tango.freedesktop.org/Tango_Desktop_Project
  License:
    Public Domain

icons/information.svg: Bobarino
  https://en.wikipedia.org/wiki/File:Information.svg
  License:
    GFDL 1.2 or later / CC-BY-SA 3.0
    https://en.wikipedia.org/wiki/File:Information.svg#Licensing

icons/add-math-channel.svg: Inductiveload
  https://en.wikipedia.org/wiki/File:Icon_Mathematical_Plot.svg
  License:
    Public Domain

QDarkStyleSheet: Colin Duquesnoy
  https://github.com/ColinDuquesnoy/QDarkStyleSheet
  License:
    CC-BY 4.0
    https://github.com/ColinDuquesnoy/QDarkStyleSheet/blob/master/LICENSE.md

DarkStyle: Juergen Skrotzky
  https://github.com/Jorgen-VikingGod/Qt-Frameless-Window-DarkStyle
  License:
    MIT license
    https://github.com/Jorgen-VikingGod/Qt-Frameless-Window-DarkStyle#licence

QHexView: Victor Anjin
  https://github.com/virinext/QHexView
  License:
    MIT license
    https://github.com/virinext/QHexView/blob/master/LICENSE

ExprTk: Arash Partow
  https://www.partow.net/programming/exprtk/index.html
  License:
    MIT license


Mailing list
------------

 https://lists.sourceforge.net/lists/listinfo/sigrok-devel


IRC
---

You can find the sigrok developers in the #sigrok IRC channel on Libera.Chat.


Website
-------

 http://sigrok.org/wiki/PulseView

## 上游

- https://sigrok.org/wiki/PulseView
- https://github.com/sigrokproject/pulseview
