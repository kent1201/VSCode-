# CMakeLists.txt 撰寫（設計）教學：Modern CMake 實戰指南

> 目標：
>
> * 用「**目標為中心（target-based）**」的 Modern CMake 寫法組織你的專案
> * 了解 `PRIVATE` / `PUBLIC` / `INTERFACE` 的傳遞邏輯
> * 會設定語言標準、編譯選項、安裝與封裝、第三方相依（Find/Fetch）
> * 提供可直接複製的範本（小／中／大型專案）

---

## 0) CMake 的核心哲學（從舊到新）

* **舊式（不推薦）**：直接操作全域變數（`CMAKE_CXX_FLAGS`、`include_directories()`）→ 影響不可控、相依混亂。
* **Modern CMake（推薦）**：**所有設定都掛在 target 上**（`target_*` APIs），並用 `PRIVATE/PUBLIC/INTERFACE` 決定傳遞性。

  * `PRIVATE`：只影響自己
  * `PUBLIC`：影響自己＋連到你的使用者
  * `INTERFACE`：只影響使用者（自己不受影響；常用在 header-only 或介面庫）

---

## 1) 最小可行範本（單一執行檔）

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloCMake CXX)

# 1) 語言標準（專案級）
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 2) 產生 compile_commands.json（IDE/工具友好）
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 3) 統一可執行檔輸出到 <repo>/bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")

# 4) 定義 target（可執行檔/函式庫）
add_executable(app src/main.cpp)

# 5) target 的編譯選項/定義/包含路徑（目標導向）
target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)
target_compile_definitions(app PRIVATE APP_VERSION="0.1.0")
target_include_directories(app PRIVATE "${CMAKE_SOURCE_DIR}/include")
```

---

## 2) `PRIVATE / PUBLIC / INTERFACE` 傳遞邏輯

假設你有一個靜態庫 `libcore` 被 `app` 連結：

```cmake
add_library(core STATIC src/core/foo.cpp)
target_include_directories(core
  PUBLIC  "${CMAKE_SOURCE_DIR}/core/include"   # app 也會看到
  PRIVATE "${CMAKE_SOURCE_DIR}/core/src")      # 只 core 自己用
target_compile_definitions(core PUBLIC USE_FAST_PATH=1)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core)  # core 的 PUBLIC/INTERFACE 設定會傳給 app
```

* `core` 的 `PUBLIC include/compile_defs` 會自動**傳遞**到 `app`，減少手動維護。

---

## 3) 常用 target 設定清單（重點 API）

```cmake
# 語言標準（若想針對單一 target 覆蓋）
target_compile_features(myexe PRIVATE cxx_std_14)

# 編譯器選項（可針對各編譯器判斷）
target_compile_options(myexe PRIVATE
  $<$<CXX_COMPILER_ID:GNU,Clang,AppleClang>:-Wall -Wextra -Wpedantic>
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->)

# 前置處理巨集
target_compile_definitions(myexe PUBLIC  USE_LIB=1)
target_compile_definitions(myexe PRIVATE INTERNAL_DEBUG=1)

# 包含路徑（PUBLIC/PRIVATE/INTERFACE 選擇傳遞）
target_include_directories(myexe PRIVATE ${CMAKE_SOURCE_DIR}/include)

# 連結函式庫
target_link_libraries(myexe PRIVATE mylib)

# 指定輸出名稱或目錄
set_target_properties(myexe PROPERTIES OUTPUT_NAME "myapp")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")
```

---

## 4) 專案結構與分層設計

**建議結構：**

```
project/
  ├─ CMakeLists.txt           # 頂層：專案設定、相依與子資料夾匯入
  ├─ cmake/                   # 自訂 FindXXX.cmake / 模組、工具
  ├─ include/                 # 對外公開的 headers
  ├─ src/                     # 主程式/共用源碼
  ├─ libs/
  │   └─ core/
  │       ├─ CMakeLists.txt   # 子模組
  │       ├─ include/core/
  │       └─ src/
  ├─ tests/                   # 測試
  └─ examples/                # 範例程式
```

**頂層 CMakeLists：**

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject VERSION 1.2.3 LANGUAGES CXX)

# 全域設定（可視需求開關）
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 選項（讓使用者可 -DENABLE_FOO=ON/OFF）
option(ENABLE_TESTS "Build tests" ON)
option(BUILD_SHARED_LIBS "Build shared instead of static" OFF)

# 子模組
add_subdirectory(libs/core)
add_subdirectory(src)

if(ENABLE_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()
```

**子模組 `libs/core/CMakeLists.txt`：**

```cmake
add_library(core src/foo.cpp src/bar.cpp)
target_include_directories(core PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include")
target_compile_features(core PUBLIC cxx_std_14)
```

**可執行檔 `src/CMakeLists.txt`：**

```cmake
add_executable(app main.cpp)
target_link_libraries(app PRIVATE core)
set_target_properties(app PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")
```

---

## 5) 找第三方函式庫：`find_package()` 與 `FetchContent`

### 5.1 `find_package()`（系統/套件管理器 已有）

以 **OpenCV** 為例：

```cmake
find_package(OpenCV 4.5 REQUIRED COMPONENTS core imgproc highgui)
target_link_libraries(app PRIVATE ${OpenCV_LIBS})
target_include_directories(app PRIVATE ${OpenCV_INCLUDE_DIRS})
```

> 需要對應的 **FindOpenCV.cmake** 或 OpenCV 安裝時提供的 CMake config。
> 若找不到：檢查 `CMAKE_PREFIX_PATH`、系統安裝路徑或提供 `-DOpenCV_DIR=...`。

### 5.2 `FetchContent`（自動抓取源碼編進來）

```cmake
include(FetchContent)
FetchContent_Declare(fmt
  GIT_REPOSITORY https://github.com/fmtlib/fmt.git
  GIT_TAG        10.2.1)
FetchContent_MakeAvailable(fmt)

target_link_libraries(app PRIVATE fmt::fmt)
```

> 優點：可重現、無外部安裝依賴；缺點：初次 configure 需要抓網、時間較長。

---

## 6) 安裝（install）、匯出（export）、封裝（package）

### 6.1 安裝你的庫與 headers

```cmake
# 庫與執行檔的安裝
install(TARGETS core app
  EXPORT myprojectTargets
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)

# 公開的頭檔
install(DIRECTORY include/ DESTINATION include)

# 匯出 targets（讓下游 find_package 使用）
install(EXPORT myprojectTargets
  NAMESPACE myproj::
  DESTINATION lib/cmake/myproject)
```

### 6.2 產生 `Config.cmake`（讓別人 `find_package(MyProject)`）

```cmake
include(CMakePackageConfigHelpers)

# 版本檔
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion)

# 簡單的 Config.cmake（可另外寫模板）
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake" "
include(\${CMAKE_CURRENT_LIST_DIR}/myprojectTargets.cmake)
")

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfigVersion.cmake"
  DESTINATION lib/cmake/myproject)
```

### 6.3 CPack 打包（.zip/.tar/.deb/.rpm…）

```cmake
include(CPack)  # 使用專案 VERSION、install 規則自動封裝
```

---

## 7) 測試：CTest + GoogleTest（例）

```cmake
# 頂層
enable_testing()
add_subdirectory(tests)
```

`tests/CMakeLists.txt`：

```cmake
include(FetchContent)
FetchContent_Declare(googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.14.0)
FetchContent_MakeAvailable(googletest)

add_executable(core_tests test_core.cpp)
target_link_libraries(core_tests PRIVATE core gtest_main)

add_test(NAME core_tests COMMAND core_tests)
```

---

## 8) Build Type、單／多配置、發佈最佳化

* **單配置**（Makefiles/Ninja）：用 `-DCMAKE_BUILD_TYPE=Release|Debug|RelWithDebInfo`
* **多配置**（Visual Studio/Xcode）：用 `--config Release` 在 build 階段指定

常見最佳化：

```cmake
# 建置類型預設
if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "" FORCE)
endif()

# 較嚴格的警告
target_compile_options(app PRIVATE
  $<$<CXX_COMPILER_ID:GNU,Clang,AppleClang>:-Wall -Wextra -Wshadow -Wconversion>
  $<$<CXX_COMPILER_ID:MSVC>:/W4 /w44265 /w44062>)

# Position Independent Code（常見於建立 shared libs）
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
```

---

## 9) 交叉編譯與工具鏈（Toolchain）快速示例

`toolchain-arm64.cmake`：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER   /opt/aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER /opt/aarch64-linux-gnu-g++)
set(CMAKE_FIND_ROOT_PATH /opt/sysroot-aarch64)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

使用：

```bash
cmake -S . -B build-arm \
  -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-arm64.cmake \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-arm -j
```

---

## 10) Generator Expressions（高級但實用）

用於**條件化**選項：

```cmake
target_compile_options(app PRIVATE
  $<$<CONFIG:Debug>:-O0 -g>
  $<$<CONFIG:Release>:-O3>
  $<$<CXX_COMPILER_ID:MSVC>:/MP>
)
```

---

## 11) 常見陷阱 & 排錯

| 問題                           | 緣由                      | 解法                                                               |
| ---------------------------- | ----------------------- | ---------------------------------------------------------------- |
| `include_directories()` 汙染全域 | 舊式寫法                    | 換 `target_include_directories()`，並用 PUBLIC/PRIVATE 控制            |
| 找不到第三方庫                      | `find_package()` 搜尋路徑不對 | 設定 `CMAKE_PREFIX_PATH`、指定 `-DXXX_DIR=...`、或改用 `FetchContent`     |
| 跨平台編譯器旗標不相容                  | 直接硬塞 `-O3 -Wall` 等      | 用 Generator Expressions 判斷編譯器                                    |
| Build type 無效                | 多配置生成器                  | 使用 `--config Release` 在 build 階段指定                               |
| RPATH/執行期找不到 dylib/so        | Linux/macOS 執行期路徑       | 研究 `CMAKE_INSTALL_RPATH`、`@rpath` 與 `patchelf/install_name_tool` |
| 大專案 Configure 慢              | 大量 Fetch 或外部檢測          | 產生並使用 **CMake Presets**、鎖定相依版本、分離 Third-party                    |

---

## 12) 小／中／大型專案範本

### A) 小型（單一執行檔）

```cmake
cmake_minimum_required(VERSION 3.16)
project(SmallApp CXX)
set(CMAKE_CXX_STANDARD 14)
add_executable(app src/main.cpp)
```

### B) 中型（執行檔 + 靜態庫 + OpenCV）

```cmake
cmake_minimum_required(VERSION 3.20)
project(MidApp VERSION 0.2.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")

add_library(core STATIC src/core/foo.cpp)
target_include_directories(core PUBLIC "${CMAKE_SOURCE_DIR}/include")
target_compile_features(core PUBLIC cxx_std_14)

find_package(OpenCV 4 REQUIRED COMPONENTS core imgproc highgui)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core ${OpenCV_LIBS})
```

### C) 大型（多模組 + 測試 + 安裝/封裝 + 選項）

```cmake
cmake_minimum_required(VERSION 3.22)
project(BigProj VERSION 1.0.0 LANGUAGES CXX)

option(ENABLE_TESTS "Build tests" ON)
option(BUILD_SHARED_LIBS "Build shared libraries" OFF)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_subdirectory(libs/core)
add_subdirectory(apps/cli)

if(ENABLE_TESTS)
  enable_testing()
  add_subdirectory(tests)
endif()

include(CPack)  # 使用安裝規則自動封裝
```

---

## 13) 與 VS Code / CMake Tools 的配合

* 在 VS Code 的 `settings.json` 加：

  ```json
  "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",
  "cmake.buildDirectory": "${workspaceFolder}/build"
  ```
* `launch.json`/`tasks.json` 建議使用 `${command:cmake.launchTargetPath}` 取得可執行檔。
* 以 **CMakeLists.txt** 為「事實來源」：**所有旗標、包含路徑、定義**都經由 target API 設置，IDE 只負責呼叫。

---

## 14) 結語

1. **一切掛在 target** 上，別再改全域旗標。
2. 清楚分層：核心庫（`PUBLIC` 介面）→ 應用程式（`PRIVATE` 相依）。
3. 第三方相依優先用 `find_package()`，找不到再 fallback 到 `FetchContent`。
4. 先做好 `install`＋`export`，未來要被 `find_package()` 或封裝會很輕鬆。
5. 與 VS Code/CMake Tools 配合時，**CMakeLists 是中心**，IDE 設定只是膠水。

