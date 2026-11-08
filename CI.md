CI and releases
===============

This document describes how the `vllogic` fork of PulseView is built and
released by GitHub Actions.  All workflows live in `.github/workflows/`.

Repositories and branches
-------------------------

| Repository                 | Branch        | Role                                   |
|----------------------------|---------------|----------------------------------------|
| `vllogic/pulseview`        | `vllogic-dev` | PulseView fork, hosts the workflows    |
| `vllogic/libsigrok`        | `vllogic-dev` | libsigrok fork with the `vllogic-logic` driver |
| `vllogic/sigrok-build`     | `vllogic-dev` | build scripts / reusable full-build workflow |

`vllogic/libsigrok` is public and is checked out directly by the workflows;
no extra credentials are needed for the normal build paths.


Workflow overview
-----------------

| File                        | Triggers                                        | Purpose |
|-----------------------------|-------------------------------------------------|---------|
| `build-multiplatform.yml`   | push `master`/`vllogic-dev`, tag `v*`, dispatch  | PulseView (4 platforms) + sigrok-cli (4 platforms) + release |
| `linux-minimal.yml`         | dispatch                                        | Historical single-job Linux x86_64 build |
| `build.yml`                 | dispatch                                        | Official full build via `vllogic/sigrok-build`, plus a tag release job |
| `qt6-probe.yml`             | dispatch                                        | Qt6 trial build (Linux + macOS), no artifacts published |

Only `build-multiplatform.yml` runs automatically.  The other three are kept
for reference or manual experiments and never publish artifacts on push.


Triggers and tag naming
-----------------------

* A push to `vllogic-dev` (or `master`) runs all eight build jobs; `release`
  is skipped because it is gated on `refs/tags/`.
* A tag matching `v*` runs the same eight build jobs and then `release`.
* `workflow_dispatch` can start any workflow manually from the Actions tab or
  with `gh workflow run <file> -R vllogic/pulseview --ref vllogic-dev`.

Release tags use:

```
v<PV_VERSION_STRING>-vllogic.<YYYYMMDD>
```

for example `v0.5.0-vllogic.20261108`.  `PV_VERSION_STRING` is defined in
`CMakeLists.txt` (currently `0.5.0`); ordinary version strings get the
`-git-<shortsha>` suffix appended by CMake, so release tags are built from a
clean tree and carry the date instead.

The tag commit must already be on `vllogic-dev`; the workflows checkout
`vllogic/pulseview` at `${{ github.ref_name }}`, so the tag itself selects the
source tree.


Build matrix
------------

### PulseView

| Platform          | Runner             | Toolchain / packages                                   | Qt      | Notes |
|-------------------|--------------------|--------------------------------------------------------|---------|-------|
| Linux x86_64      | `ubuntu-22.04`     | apt `build-essential`, autotools, `doxygen`, glib/glibmm, libusb, libzip, libserialport, boost | Qt5 (`qtbase5-dev`) | baseline glibc 2.35 |
| Linux arm64       | `ubuntu-24.04-arm` | same apt package set                                  | Qt5     | native arm64 runner |
| Windows x86_64    | `windows-2022`     | MSYS2 **UCRT64**, `mingw-w64-ucrt-x86_64-*` packages, MSYS `doxygen` | Qt5 (`qt5-base`/`qt5-svg`/`qt5-tools`) | see Windows notes |
| macOS arm64       | `macos-15`         | Homebrew: `qt`, `glibmm`, `libsigc++`, `libusb`, `libzip`, `boost`, `libserialport`, autotools, `doxygen` | **Qt6** (`brew install qt`) | see macOS notes |

All platforms build libsigrok from source with:

```
./configure --prefix=<sr> \
  --enable-cxx --disable-python --disable-ruby --disable-java \
  --disable-all-drivers --enable-vllogic-logic
```

so the resulting binaries only contain the `vllogic-logic` driver.  PulseView
is configured with `-DENABLE_DECODE=OFF` (no libsigrokdecode) and
`-DCMAKE_POLICY_VERSION_MINIMUM=3.5` (needed for CMake 4.x).

### sigrok-cli

The same four targets are built as separate jobs (`sigrok-cli-linux` matrix,
`sigrok-cli-macos-arm64`, `sigrok-cli-windows-x86_64`):

1. build/install libsigrok + C++ bindings into `<workspace>/sr`;
2. `git clone --depth 1 https://github.com/sigrokproject/sigrok-cli.git`;
3. `./autogen.sh`, then
   `PKG_CONFIG_PATH=<workspace>/sr/lib/pkgconfig ./configure --prefix=<workspace>/scli --without-libsigrokdecode`,
   `make -j`, `make install`;
4. package the binary plus `libsigrok`/`libsigrokcxx`.

sigrok-cli itself is a C program and does not need Qt; on Windows the UCRT64
runtime DLLs are collected with `ldd`.


Platform specific steps
-----------------------

### Linux

* Qt5 and all dependencies come from apt.
* `PKG_CONFIG_PATH` must contain both `<workspace>/sr/lib/pkgconfig` and the
  system directories so PulseView finds `libsigrokcxx`.
* Packaging copies `libsigrok.so*`, `libsigrokcxx.so*` and a
  `run-pulseview.sh` / `run-sigrok-cli.sh` that sets `LD_LIBRARY_PATH`.

### Windows (MSYS2 UCRT64)

* `msystem: UCRT64`, package prefix `mingw-w64-ucrt-x86_64-`.
* `doxygen` comes from the MSYS2 *msys* repository (POSIX build), not from
  `mingw-w64-*`; the native `doxygen.exe` cannot read the POSIX `SRCDIR`
  paths passed by libsigrok's `Makefile.am`, and fails with
  `doxy/xml/index.xml Error 3`.
* Qt deployment uses `windeployqt-qt5` (MSYS2 suffixes Qt5 tools with `-qt5`)
  and **`--no-angle`**: MSYS2's Qt5 does not ship `libGLESv2.dll`/`libEGL.dll`.
  PulseView is a QWidget/QPainter application and never loads ANGLE, so this
  is safe; it would only matter if Qt Quick/3D were introduced.
* Remaining runtime DLLs are collected with `ldd` filtered on `/ucrt64/`.
* A `sigrok-cli.exe -V` / `pulseview.exe -V` smoke test runs on the runner.

### macOS (arm64)

* PulseView is built against **Qt6** from Homebrew (`brew install qt`), which
  removed the deprecated `qt@5` warning.  `CMakeLists.txt` already supports
  both Qt5 and Qt6; no source changes are needed.
* `PKG_CONFIG_PATH` includes `$(brew --prefix qt)/lib/pkgconfig`,
  `$(brew --prefix)/lib/pkgconfig` and `<workspace>/sr/lib/pkgconfig`.
* `libsigrok.4.dylib` / `libsigrokcxx.4.dylib` are copied next to the binary
  and rewritten with `install_name_tool` to `@loader_path/...`; the binaries
  are then re-signed ad-hoc with `codesign --force --sign -` (required after
  modifying a Mach-O file on Apple Silicon; no certificate is involved).
* The bundle still expects Homebrew Qt6 at runtime.


Artifacts
---------

Every build job uploads a **plain folder** (no archive inside the artifact):

```
pulseview-linux-x86_64/       pulseview-linux-arm64/
pulseview-windows-x86_64/     pulseview-macos-arm64/
sigrok-cli-linux-x86_64/      sigrok-cli-linux-arm64/
sigrok-cli-windows-x86_64/    sigrok-cli-macos-arm64/
```

Downloading an artifact and unzipping it therefore yields the
`pulseview-<platform>/` (or `sigrok-cli-<platform>/`) directory directly,
containing the executable, the bundled libsigrok libraries and the run
script.

Note: GitHub's `upload-artifact` does not preserve file modes, so files in
artifacts arrive with mode 0644.  The release job restores the executable
bits before archiving.


Release flow
------------

`release` runs only for tags and depends on all eight build jobs
(`needs: [linux, macos-arm64, windows-x86_64, sigrok-cli-linux,
sigrok-cli-macos-arm64, sigrok-cli-windows-x86_64]`, where `linux` and
`sigrok-cli-linux` are matrices).

Steps:

1. `actions/download-artifact@v8` with `path: dist` and
   `merge-multiple: true` — the eight folders land as
   `dist/<artifact-name>/...` without an extra per-artifact directory.
2. `chmod +x` on `pulseview`, `sigrok-cli` and `run-*.sh` (executable bits
   are lost during artifact upload).
3. Create the eight single-level archives:

   ```
   tar -czf pulseview-linux-x86_64.tar.gz -C dist pulseview-linux-x86_64
   tar -czf pulseview-linux-arm64.tar.gz  -C dist pulseview-linux-arm64
   tar -czf pulseview-macos-arm64.tar.gz  -C dist pulseview-macos-arm64
   (cd dist && zip -r ../pulseview-windows-x86_64.zip pulseview-windows-x86_64)
   tar -czf sigrok-cli-linux-x86_64.tar.gz -C dist sigrok-cli-linux-x86_64
   tar -czf sigrok-cli-linux-arm64.tar.gz  -C dist sigrok-cli-linux-arm64
   tar -czf sigrok-cli-macos-arm64.tar.gz  -C dist sigrok-cli-macos-arm64
   (cd dist && zip -r ../sigrok-cli-windows-x86_64.zip sigrok-cli-windows-x86_64)
   ```

4. `softprops/action-gh-release@v2` uploads exactly those eight files and
   generates release notes.

Resulting assets: PulseView 4 + sigrok-cli 4.

To redo a release:

```
gh release delete <tag> -R vllogic/pulseview --yes
git push origin :refs/tags/<tag>
git tag -d <tag>
# fix/push vllogic-dev if needed, then
git tag <tag> && git push origin <tag>
```


Known limitations
-----------------

* **Linux packages target Ubuntu 22.04.** The binaries link against
  `libzip.so.4`, glib and Boost from 22.04; on newer distributions (24.04+
  ships `libzip.so.5`) they need a compatibility package, or a future switch
  to AppImage/static linking.
* **Windows needs libusbK** for the dual-interface device; fully driver-free
  operation depends on future firmware support.
* **macOS package depends on `brew install qt`** (Qt6).  Qt5 is no longer
  used on macOS.
* **Package file modes in artifacts** are 0644 (GitHub limitation); release
  archives are fixed up.
* **macOS arm64 runners** can queue for a long time (GitHub capacity notice).
* `softprops/action-gh-release@v2` still targets Node.js 20 and triggers a
  deprecation annotation (harmless, GitHub runs it on Node 24).
* `build.yml` calls the full official `vllogic/sigrok-build` matrix and is
  dispatch-only; it is not part of the release path.


Local reproduction
------------------

libsigrok (prefix `/tmp/sigrok-vllogic`):

```
./autogen.sh
./configure --prefix=/tmp/sigrok-vllogic \
  --enable-cxx --disable-python --disable-ruby --disable-java \
  --disable-all-drivers --enable-vllogic-logic
make -j"$(nproc)" && make install
```

PulseView against an installed libsigrok in `$PREFIX`:

```
export PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig"
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH="$PREFIX" \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DENABLE_DECODE=OFF
cmake --build build -j"$(nproc)"
QT_QPA_PLATFORM=offscreen ./build/pulseview -V
```

`sigrok-cli` (same prefix):

```
git clone --depth 1 https://github.com/sigrokproject/sigrok-cli.git
cd sigrok-cli
./autogen.sh
PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig" ./configure \
  --prefix="$PREFIX" --without-libsigrokdecode
make -j"$(nproc)" && make install
```

Version
-------

The user-visible version is `PV_VERSION_STRING` in `CMakeLists.txt`
(currently `0.5.0`).  Builds from git add `-git-<shortsha>`; release tags use
`v<version>-vllogic.<YYYYMMDD>` as described above.
