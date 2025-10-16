# CMakeLists.txt 極簡基礎教學（逐行解釋版）

## 0) CMake 是什麼？基本觀念先釐清

* **CMake** 不是編譯器；它是「**產生建置系統**」的工具（例如 Makefiles、Ninja、Visual Studio 專案檔）。流程通常是：
  **Configure/Generate → Build**。你用 `cmake` 讀 `CMakeLists.txt` 產生建置系統，再用 `cmake --build` 或本地 build 工具去編譯。([cmake.org][1])
* 官方有一個循序漸進的 Tutorial，從「最小專案」到「加庫、安裝、測試」都有。本文的做法與它一致，只是更白話。([cmake.org][2])

---

## 1) 先做一個最小可跑的專案

### 專案結構

```
hello/
├─ CMakeLists.txt
└─ src/
   └─ main.cpp
```

`src/main.cpp`

```cpp
#include <iostream>
int main() { std::cout << "Hello CMake!\n"; }
```

`CMakeLists.txt`（逐行說明）

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloCMake LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(hello src/main.cpp)
```

**每行在做什麼：**

1. `cmake_minimum_required(VERSION 3.16)`
   告訴 CMake：本專案**至少**要用 3.16 以上來設定；同時也會把「相容性政策（policies）」調整到這版的行為。版本太舊就直接報錯，避免不同版本行為不一致。([cmake.org][3])
2. `project(HelloCMake LANGUAGES CXX)`
   宣告專案名稱與啟用的語言（這裡是 C++）。`project()` 也會初始化一些變數（例如 `PROJECT_SOURCE_DIR`、版本資訊…）。**慣例：**放在 `cmake_minimum_required()` 之後、其他設定之前。([cmake.org][4])
3. `set(CMAKE_CXX_STANDARD 14)`／`set(CMAKE_CXX_STANDARD_REQUIRED ON)`／`set(CMAKE_CXX_EXTENSIONS OFF)`
   指定使用 **C++14**，而且是**必須**（不是隨緣），同時關閉編譯器的方言延伸（如 GNU++14），確保可攜性。這做法也見於官方 Tutorial 的「指定 C++ 標準」章節。([cmake.org][2])
4. `add_executable(hello src/main.cpp)`
   建立一個可執行檔「目標」（target）叫 `hello`，來源檔是 `src/main.cpp`。**注意**：目標名稱在專案中要唯一。之後你對 `hello` 加的旗標、包含路徑，全部用「target_*」API 綁在它身上。([cmake.org][5])

### 怎麼建置與執行

```bash
# 在專案根目錄
cmake -S . -B build        # Configure/Generate（把建置檔生成到 build/）
cmake --build build -j     # Build（呼叫對應的本地編譯系統）
./build/hello              # 執行（Windows 依生成器、組態而異）
```

* 在 **Windows 的多組態生成器（如 Visual Studio）** 情況下，Release/Debug 是在 build 階段指定：
  `cmake --build build --config Release`。([cmake.org][6])

---

## 2) 再加一點點：把共用程式做成「庫」並連結（認識 PUBLIC/PRIVATE）

### 結構

```
hello_lib/
├─ CMakeLists.txt
├─ include/
│  └─ util/add.h
├─ src/
│  └─ add.cpp
└─ app/
   └─ main.cpp
```

`include/util/add.h`

```cpp
#pragma once
int add(int a, int b);
```

`src/add.cpp`

```cpp
#include "util/add.h"
int add(int a, int b){ return a + b; }
```

`app/main.cpp`

```cpp
#include <iostream>
#include "util/add.h"
int main(){ std::cout << add(3,4) << "\n"; }
```

`CMakeLists.txt`（逐行說明）

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloLib LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 1) 建靜態庫 target：util
add_library(util STATIC src/add.cpp)

# 2) 公開頭檔目錄
target_include_directories(util PUBLIC ${CMAKE_SOURCE_DIR}/include)

# 3) 可執行檔 app，並連結到 util
add_executable(app app/main.cpp)
target_link_libraries(app PRIVATE util)
```

**關鍵概念：**

* `target_include_directories(util PUBLIC …)`：把 `include/` 宣告成 **PUBLIC**「使用需求（usage requirement）」。任何連到 `util` 的目標（例如 `app`）會**自動繼承**這個 include 路徑；你就不用在 `app` 再寫一次。這就是**目標導向（target-based）**寫法的好處。([cmake.org][7])
* `target_link_libraries(app PRIVATE util)`：`app` 連 `util`。因為 `util` 對外公開了 `include/`（PUBLIC），所以 `app` 可直接 `#include "util/add.h"`。
  （**直覺記法**：PRIVATE=只影響自己；PUBLIC=自己+下游；INTERFACE=只影響下游。）([cmake.org][7])

---

## 3) 找第三方（最小例：OpenCV）

如果系統已安裝好 OpenCV 且有對應的 `OpenCVConfig.cmake`，你可以這樣寫：

```cmake
find_package(OpenCV 4 REQUIRED COMPONENTS core imgproc highgui)
add_executable(show app/show.cpp)
target_link_libraries(show PRIVATE ${OpenCV_LIBS})
target_include_directories(show PRIVATE ${OpenCV_INCLUDE_DIRS})
```

* `find_package(OpenCV 4 REQUIRED …)`：要求找到 OpenCV（至少 4.x）。找不到就報錯。([cmake.org][8])
* 如果找不到，通常是搜尋路徑不對；可用 `-DOpenCV_DIR=...` 或設 `CMAKE_PREFIX_PATH`。`find_package()` 的搜尋行為與參數是固定規則。([cmake.org][9])

---

## 4) 最少會用到的「命令」與「變數」

### 常見命令（會一直用到）

* `cmake_minimum_required(VERSION ...)`：限制 CMake 版本並設定相容性政策。([cmake.org][3])
* `project(NAME LANGUAGES ...)`：設定專案名稱、語言、（可選）版本。([cmake.org][4])
* `add_executable(name sources...)`：建立可執行檔目標。([cmake.org][5])
* `add_library(name [STATIC|SHARED] sources...)`：建立函式庫目標。
* `target_include_directories(tgt [PRIVATE|PUBLIC|INTERFACE] dirs...)`：設定包含路徑（綁在目標上）。
* `target_link_libraries(tgt [PRIVATE|PUBLIC|INTERFACE] deps...)`：連結相依（綁在目標上）。
* `target_compile_options/definitions/features(...)`：編譯旗標、巨集、語言標準（綁在目標上）。
* `find_package(Pkg ...)`：找外部套件（系統/套件管理器已安裝或提供 Config）。([cmake.org][9])

### 常見變數（你會看到）

* `PROJECT_SOURCE_DIR`：頂層 `CMakeLists.txt` 所在目錄。
* `PROJECT_BINARY_DIR`：你 `-B` 指到的 build 目錄。
* `CMAKE_CXX_STANDARD` / `CMAKE_CXX_STANDARD_REQUIRED` / `CMAKE_CXX_EXTENSIONS`：全域設定 C++ 標準與嚴格度。這也可用 `target_compile_features(... cxx_std_14)` 針對**單一目標**設定。([cmake.org][2])

---

## 5) 在 VS Code / WSL 裡的最短路徑

* 安裝 **CMake Tools** 與 **MS C/C++** 擴充，然後 `F1 → CMake: Quick Start` 產生骨架。
* 之後底部欄位就能 Configure/Build/Debug；在 WSL 也一樣可以直接用。([Microsoft Learn][10])

---

## 6) 常見新手小坑（一句話對策）

| 狀況                        | 為什麼                           | 快解                                                                                                      |
| ------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------- |
| `#include` 找不到            | 沒把 include 掛在**提供方 target** 上 | 對「庫」用 `target_include_directories(core PUBLIC include/)`，下游自動繼承。([cmake.org][7])                        |
| Release/Debug 沒效果         | 你用的是 VS（多組態生成器）               | build 時加 `--config Release`；單組態（Make/Ninja）則在 configure 加 `-DCMAKE_BUILD_TYPE=Release`。([cmake.org][6]) |
| `find_package(OpenCV)` 失敗 | 沒有找到 *Config.cmake* 或路徑不在搜尋範圍 | 指定 `-DOpenCV_DIR=` 或 `-DCMAKE_PREFIX_PATH=`。([cmake.org][8])                                            |
| 目標沒吃到旗標                   | 旗標塞在全域或塞錯地方                   | 一律用 `target_*` 綁目標（Modern CMake 習慣）。([cmake.org][7])                                                    |

---

## 7) 兩個你可以立刻複製的最小模板

### (1) 單一執行檔（C++14）

```cmake
cmake_minimum_required(VERSION 3.16)
project(SmallApp LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
add_executable(app src/main.cpp)
```

### (2) 庫 + 執行檔（含 PUBLIC include）

```cmake
cmake_minimum_required(VERSION 3.16)
project(MidApp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_library(core STATIC src/core/foo.cpp)
target_include_directories(core PUBLIC ${CMAKE_SOURCE_DIR}/include)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core)
```

---

## 8) 想更進一步？

* 官方 Tutorial（一步一步照做）→ 安裝、測試、產生封裝…都從這裡開始。([cmake.org][2])
* `cmake` 指令手冊（怎麼正確下 `cmake -S -B`、`cmake --build`）：([cmake.org][6])
* `project()`、`add_executable()`、`cmake_minimum_required()` 的正式文件（遇到細節就翻）：([cmake.org][4])

---

[1]: https://cmake.org/cmake/help/latest/index.html?utm_source=chatgpt.com "CMake 4.1.2 Documentation"
[2]: https://cmake.org/cmake/help/latest/guide/tutorial/index.html?utm_source=chatgpt.com "CMake Tutorial — CMake 4.1.2 Documentation"
[3]: https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html?utm_source=chatgpt.com "cmake_minimum_required — CMake 4.1.2 Documentation"
[4]: https://cmake.org/cmake/help/latest/command/project.html?utm_source=chatgpt.com "project — CMake 4.1.2 Documentation"
[5]: https://cmake.org/cmake/help/latest/command/add_executable.html?utm_source=chatgpt.com "add_executable — CMake 4.1.2 Documentation"
[6]: https://cmake.org/cmake/help/latest/manual/cmake.1.html?utm_source=chatgpt.com "cmake(1) — CMake 4.1.2 Documentation"
[7]: https://cmake.org/cmake/help/latest/guide/tutorial/A%20Basic%20Starting%20Point.html?utm_source=chatgpt.com "Step 1: A Basic Starting Point"
[8]: https://cmake.org/cmake/help/book/mastering-cmake/cmake/Help/guide/tutorial/?utm_source=chatgpt.com "CMake Tutorial"
[9]: https://cmake.org/cmake/help/v3.5/command/add_executable.html?utm_source=chatgpt.com "add_executable — CMake 3.5.2 Documentation"
[10]: https://learn.microsoft.com/en-us/cpp/build/cmake-projects-in-visual-studio?view=msvc-170&utm_source=chatgpt.com "CMake projects in Visual Studio"
