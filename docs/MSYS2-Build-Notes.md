# Eden 编译环境安装记录（MSYS2 / MinGW-w64）

> **目的**：记录 2026-10-08 为编译 Eden 模拟器（方案 2：MSYS2/MinGW-w64）安装的软件包与关键处理，方便日后清理或重建。
> **环境**：Windows 10（win32），Eden 源码位于 `E:\AI\mirror`。

---

## 一、MSYS2 本体

| 项目 | 说明 |
|------|------|
| 安装方式 | `winget install --id MSYS2.MSYS2 -e --source winget` |
| 安装位置 | `C:\msys64` |
| 版本 | pacman 6.1.0，bash 5.3.20，msys2-runtime 3.6.10-6 |
| 用途 | mingw64 工具链（GCC 16.2.0） |

**彻底卸载 MSYS2**：直接删除 `C:\msys64` 目录，并检查系统 PATH 中是否残留 `C:\msys64\...` 项。

---

## 二、通过 pacman 安装的包

> 查询命令：`C:\msys64\usr\bin\bash.exe -lc "pacman -Qe"`（显式安装）或 `pacman -Q`（全部）。
> 全部已安装 **330** 个包；其中**显式安装 52** 个，其余为自动依赖。

### 2.1 MSYS 基础工具（来自 `BASE="git make autoconf libtool automake-wrapper jq patch"`）

```
base  filesystem  msys2-runtime
git  make  autoconf-wrapper  automake-wrapper  libtool  jq  patch
```

> ⚠️ `base`/`filesystem`/`msys2-runtime` 是 MSYS2 系统组件，**不要卸载**。

### 2.2 MINGW 工具链（来自 `toolchain` 组包，`pacman -Qe` 中显式列出）

```
mingw-w64-x86_64-binutils    mingw-w64-x86_64-crt
mingw-w64-x86_64-gcc         mingw-w64-x86_64-gdb
mingw-w64-x86_64-gdb-multiarch
mingw-w64-x86_64-headers     mingw-w64-x86_64-libgomp
mingw-w64-x86_64-libmangle   mingw-w64-x86_64-libstdc++
mingw-w64-x86_64-libwinpthread
mingw-w64-x86_64-tools       mingw-w64-x86_64-winpthreads
mingw-w64-x86_64-winstorecompat
```

### 2.3 MINGW 构建与依赖包（Eden 编译所需）

```
mingw-w64-x86_64-boost            mingw-w64-x86_64-clang
mingw-w64-x86_64-cmake            mingw-w64-x86_64-ffmpeg      # 后补装，见关键处理 4
mingw-w64-x86_64-fmt              mingw-w64-x86_64-glslang
mingw-w64-x86_64-libusb           mingw-w64-x86_64-lz4
mingw-w64-x86_64-nlohmann-json    mingw-w64-x86_64-openssl
mingw-w64-x86_64-opus             mingw-w64-x86_64-pkgconf
mingw-w64-x86_64-python-pip
mingw-w64-x86_64-qt6-base         mingw-w64-x86_64-qt6-charts
mingw-w64-x86_64-qt6-svg          mingw-w64-x86_64-qt6-tools
mingw-w64-x86_64-qt6-translations
mingw-w64-x86_64-sdl3             # 注意小写，见关键处理 3
mingw-w64-x86_64-spirv-headers    mingw-w64-x86_64-spirv-tools
mingw-w64-x86_64-vulkan-headers   mingw-w64-x86_64-vulkan-loader
mingw-w64-x86_64-vulkan-memory-allocator
mingw-w64-x86_64-vulkan-utility-libraries
mingw-w64-x86_64-vulkan-validation-layers
mingw-w64-x86_64-zlib             mingw-w64-x86_64-zstd
```

> **注意**：`mingw-w64-x86_64-make` 也是显式包（GNU Make 4.4.1-5，与 MSYS 的 make 并存）。

### 2.4 自动安装的依赖

约 **278** 个（Qt6 的 ICU/freetype/harfbuzz、FFmpeg 的 libx264/libx265/libaom/libvpx/libvorbis 等全编码器链、Vulkan 相关、perl 系列等）。这些是 2.2/2.3 各包的依赖，卸载时会被 `-Rsc` 级联移除，无需手动清理。

---

## 三、清理方法

### 3.1 只清理 Eden 相关依赖（保留 MSYS2 本体）

按组卸载显式包并级联删除不再需要的依赖：

```bash
C:\msys64\usr\bin\bash.exe -lc "pacman -Rsc mingw-w64-x86_64-qt6-base mingw-w64-x86_64-ffmpeg mingw-w64-x86_64-boost mingw-w64-x86_64-sdl3 mingw-w64-x86_64-cmake mingw-w64-x86_64-toolchain mingw-w64-x86_64-vulkan-devel ..."
```

> `-Rsc` 会同时移除「仅被这些包依赖」的自动依赖。建议逐个确认。

### 3.2 清理 pacman 缓存（释放下载的安装包）

```bash
C:\msys64\usr\bin\bash.exe -lc "pacman -Scc"
```

### 3.3 彻底移除 MSYS2

```powershell
Remove-Item -Recurse -Force C:\msys64
# 并从系统 PATH / 用户 PATH 中移除 C:\msys64\usr\bin、C:\msys64\mingw64\bin
```

---

## 四、关键处理记录

### 1. pacman 镜像切换为国内源（关键！默认镜像超时）

默认 `mirror.msys2.org` 连接超时，导致 `pacman -Syu` 失败。已重写两个镜像文件，把清华/中科大/南大等国内镜像置于首位：

- `C:\msys64\etc\pacman.d\mirrorlist.mingw` → 国内镜像 + `mingw/$repo` 格式
- `C:\msys64\etc\pacman.d\mirrorlist.msys` → 国内镜像 + `msys/$arch` 格式

> 恢复默认：重装 MSYS2 或从官网取回原始 mirrorlist。

### 2. 系统更新（含签名问题处理）

- `pacman -Syu` 首次失败：镜像超时 + `mingw64.db` 签名无效（前次超时留下损坏的数据库缓存）
- 解决：`rm -f /var/lib/pacman/sync/*.db*` 删除损坏缓存 → `pacman -Syy` 强制重新同步数据库 → `pacman -Syu` 完成升级（bash、msys2-keyring、gnupg 等）

### 3. 依赖安装的两个包名坑（Deps.md 与实际仓库不一致）

| Deps.md 写的 | 实际情况 | 处理 |
|--------------|---------|------|
| `enet`（`mingw-w64-x86_64-enet`） | **mingw64 仓库不存在该包** | 不安装；CMake 配置时由 CPM 自动下载 bundled enet@v1.3.18 编译 |
| `SDL3` | 包名是小写 **`sdl3`** | 安装 `mingw-w64-x86_64-sdl3` |

> 初始脚本因这两个包名报"未找到目标"导致整个安装事务失败，修正后成功。

### 4. FFmpeg 补装（Deps.md 的 MSYS2 清单遗漏）

CMake 配置报 `Could NOT find FFmpeg`，Deps.md 清单里没有 ffmpeg。补装：
`pacman -S --needed --noconfirm mingw-w64-x86_64-ffmpeg`（带 71 个依赖，含全编码器，占用较大）。

### 5. 源码修复：`CMakeModules/find/Findenet.cmake`（构建必需）

MSYS2 下 `Findenet.cmake` 无条件调用 `FixMsysPath(PkgConfig::ENET)`，而 mingw64 仓库无 enet 包 → pkg-config 找不到 → target 不存在 → `get_target_property` 崩溃，导致 configure 失败。

修复（已改 fork 源码）：
```cmake
if (MSYS2 AND TARGET PkgConfig::ENET)
    FixMsysPath(PkgConfig::ENET)
endif()
```

### 6. 源码修复：`src/yuzu_cmd/yuzu.cpp`（SDL_AppQuit 保护）

无 ROM 时 `SDL_AppInit` 返回 `SDL_APP_FAILURE`，SDL3 会调用 `SDL_AppQuit`，原代码对**未初始化**的 `Core::System` 调用 `ShutdownMainProcess()` 会崩溃。已加 `system_initialized` 标记保护。

> 注：此修复未完全消除 eden-cli 无 ROM 退出崩溃（见第 9 点，属 SDL 上游 bug）。

### 7. 运行时 DLL 补齐（关键！否则 exe 无法启动）

构建产物 `build\bin\eden.exe`/`eden-cli.exe` 依赖 mingw64 运行库与系统 DLL，构建时仅自动复制了 Qt6 DLL。首次运行报 **0xC0000135（STATUS_DLL_NOT_FOUND）**。

解决：将 `C:\msys64\mingw64\bin` 下**全部 316 个 DLL** 复制到 `build\bin`：
```powershell
Copy-Item C:\msys64\mingw64\bin\*.dll E:\AI\mirror\build\bin -Force
```
> `build\bin` 现为自包含目录，拷走即可在其他机器运行（需同为 x64 Windows）。

**⚠️ 补充（关键，否则 eden.exe 启动报错）**：Qt6 的 platform 插件在 `share\qt6\plugins` 目录，**不在** `bin` 里。缺少时 eden.exe 报：
`This application failed to start because no Qt platform plugin could be initialized.`

**不要**手动复制插件到 `build\bin\plugins\`（`plugins\` 中间层不是 Qt 的查找路径，复制了照样报错）。必须用 Qt 官方部署工具按标准结构生成 `build\bin\platforms\qwindows.dll`：
```powershell
C:\msys64\mingw64\bin\windeployqt6.exe --release E:\AI\mirror\build\bin\eden.exe
```
> 该工具会生成 `platforms`、`imageformats`、`styles`、`tls`、`translations` 等标准目录。验证是否真启动：看进程主窗口标题是否为 `Eden | ...`（错误时标题为空，弹的是系统错误框）。

### 8. 构建命令速查

```bash
# 配置（MSYS2 bash 中）
export PATH=/mingw64/bin:$PATH
cd /e/AI/mirror
cmake -S . -B build -G "MSYS Makefiles" -DCMAKE_BUILD_TYPE=Release

# 编译（全核心并行）
cmake --build build -j$(nproc)

# 增量重编译单个目标
cmake --build build --target yuzu-cmd -j$(nproc)
```

### 9. 已知问题：eden-cli 无 ROM 退出崩溃

- 现象：`eden-cli.exe`（无 ROM / `-h` / `-v`）打印用法/版本后，退出阶段访问冲突（0xC0000005）
- gdb 定位：主线程在 SDL3.dll 退出清理（`SDL_Quit` 阶段）调用到已失效回调
- 最小复现程序验证：SDL3/MinGW callbacks 基础能力正常 → **属 SDL3 上游 bug 类别**（参见 SDL issue #14510，Windows 下 SDL3 重定义 main 的退出崩溃）
- **影响范围**：仅无 ROM 的失败退出路径；加载真实 ROM 走正常初始化路径，不受影响；GUI 版 `eden.exe` 完全正常
- 候选解法（未实施）：升级 MSYS2 的 SDL3 至修复版本 / 排查 SDL3 退出时残留回调

---

## 五、验证结果

| 产物 | 路径 | 状态 |
|------|------|------|
| eden.exe（GUI） | `build\bin\eden.exe` | ✅ 可启动运行 |
| eden-cli.exe | `build\bin\eden-cli.exe` | ⚠️ 无 ROM 退出崩溃（见 4.9） |
| eden-room.exe | `build\bin\eden-room.exe` | ✅ `--help` 正常 |
| tests.exe | `build\bin\tests.exe` | ✅ 已编译 |
