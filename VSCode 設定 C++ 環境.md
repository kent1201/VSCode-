---

# C++ 專案：VS Code + CMake 從零安裝到編譯除錯

> 目標：
>
> * 在 Windows / macOS / Linux 安裝 C++ 編譯環境
> * 設定 VS Code（含 CMake Tools 與 Workspace）
> * 成功 **Configure → Build → Run/Debug**
> * 了解這份 Workspace 在做什麼、以及常見錯誤如何排除

---

## 0. 先決條件檢查

請先確認以下工具（無則略過，下一節會教你安裝）：

```bash
cmake --version
g++ --version        # 或 clang++ --version / cl.exe（MSVC）
gdb --version        # macOS 使用 lldb --version
ninja --version      # 若要用 Ninja（可選）
```

---

## 1. 安裝指南

### Windows（兩種路徑擇一）

**路徑 A：MSYS2 / MinGW（建議給 GCC/MinGW 使用者）**

1. 安裝 [MSYS2]，打開「MSYS2 MinGW x64」終端機，執行：

   ```bash
   pacman -Syu          # 先更新，提示後關閉重開，再執行一次
   pacman -S --needed mingw-w64-x86_64-{gcc,gdb,cmake,ninja,make}
   ```
2. 把 `C:\msys64\mingw64\bin` 加入 **系統 PATH**。

**路徑 B：Visual Studio Build Tools（MSVC 工具鏈）**

1. 安裝 **Visual Studio Build Tools**（勾選「C++ Desktop development」）。
2. gdb 不用裝；VS Code 會用 `cppvsdbg` 除錯器。

> 若你想使用 WSL2（Ubuntu 等），流程與 Linux 類似，編譯更接近 Linux 環境。

### macOS

```bash
xcode-select --install         # 安裝 Command Line Tools（含 clang/lldb）
brew install cmake ninja gdb   # gdb 可選（macOS 預設用 lldb）
```

### Linux（Debian/Ubuntu 範例）

```bash
sudo apt update
sudo apt install -y build-essential cmake ninja-build gdb clang lldb
```

---

## 2. 安裝 VS Code 與擴充套件

1. 安裝 **Visual Studio Code**。
2. 安裝以下 extensions：

   * `ms-vscode.cpptools`
   * `ms-vscode.cpptools-themes`（可選）
   * `ms-vscode.cmake-tools`

---

## 3. 建立一個最小可編譯專案

目錄結構（建議）：

```
your_project/
  ├─ src/
  │   └─ main.cpp
  ├─ CMakeLists.txt
  └─ your_project.code-workspace
```

`src/main.cpp`：

```cpp
#include <iostream>
int main() {
  std::cout << "Hello, VS Code + CMake!\n";
  return 0;
}
```

`CMakeLists.txt`（最小可行，並把可執行檔輸出到 `bin/`）：

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloVSCode CXX)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 可執行檔輸出到 <repo>/bin
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/bin")

add_executable(main src/main.cpp)
```

---

## 4. Workspace（.code-workspace）設定

把以下內容存成 `your_project.code-workspace`（檔名可自訂），放在專案根目錄：

```jsonc
{
  // https://code.visualstudio.com/docs/editor/workspaces

  "folders": [
    { "path": "." }
  ],

  // 推薦安裝的 extensions（會跳提示）
  "extensions": {
    "recommendations": [
      "ms-vscode.cpptools",
      "ms-vscode.cpptools-themes",
      "ms-vscode.cmake-tools"
    ],
    "unwantedRecommendations": []
  },

  // 全域設定（此 Workspace 有效）
  "settings": {
    "files.trimTrailingWhitespace": true,
    "files.insertFinalNewline": true,
    "files.trimFinalNewlines": true,

    // 打開專案就先 CMake Configure
    "cmake.configureOnOpen": true,

    // 讓 IntelliSense 直接讀 CMake 產生的 compile_commands.json
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",

    // （可選）固定 build 目錄（更可預期）
    "cmake.buildDirectory": "${workspaceFolder}/build",

    // 你專案的副檔名語言聯想（可自行擴充）
    "files.associations": {
      "glob.h": "c",
      "lvgl.h": "c"
    }
  },

  // 任務（Build / Build and Run）
  "tasks": {
    "version": "2.0.0",
    "tasks": [
      {
        "label": "Build",
        "command": "cmake",
        "args": [
          "--build", "${command:cmake.buildDirectory}"
        ],
        "group": { "kind": "build", "isDefault": true },
        "problemMatcher": {
          "owner": "cpp",
          "fileLocation": ["relative", "${workspaceFolder}"],
          "pattern": {
            "regexp": "^(.*):(\\d+):(\\d+):\\s+(warning|error):\\s+(.*)$",
            "file": 1, "line": 2, "column": 3, "severity": 4, "message": 5
          }
        }
      },
      {
        "label": "Build and Run",
        "type": "shell",
        // 使用 CMake Tools 提供的 launchTargetPath（無論輸出在哪都能找到）
        "command": "${command:cmake.launchTargetPath}",
        "group": { "kind": "test", "isDefault": true },
        "dependsOn": "Build"
      }
    ]
  },

  // 除錯設定（Linux/MinGW: gdb；macOS: lldb；Windows MSVC: cppvsdbg）
  "launch": {
    "version": "0.2.0",
    "configurations": [
      {
        "name": "Debug (gdb/lldb via CMake)",
        "type": "cppdbg",
        "request": "launch",
        "program": "${command:cmake.launchTargetPath}",
        "args": [],
        "cwd": "${workspaceFolder}",
        "preLaunchTask": "Build",
        "stopAtEntry": false,

        // Linux / MinGW 預設走 gdb；macOS 走 lldb
        "linux": { "MIMode": "gdb", "miDebuggerPath": "/usr/bin/gdb" },
        "osx":   { "MIMode": "lldb" },

        // Windows - 若你用 MinGW/gdb
        "windows": {
          "MIMode": "gdb",
          "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe"
        }
      },

      // Windows - 若使用 MSVC（Visual Studio Build Tools）
      {
        "name": "Debug (MSVC / cppvsdbg)",
        "type": "cppvsdbg",
        "request": "launch",
        "program": "${command:cmake.launchTargetPath}",
        "args": [],
        "cwd": "${workspaceFolder}",
        "preLaunchTask": "Build",
        "stopAtEntry": false
      }
    ]
  }
}
```

> 重點：
>
> * `Build` 任務呼叫 `cmake --build …`，讓 CMake 決定實際編譯器/旗標。
> * `Build and Run` / `Debug` 都用 `${command:cmake.launchTargetPath}`，不用擔心可執行檔實際輸出路徑。
> * `C_Cpp.default.configurationProvider` 讓 IntelliSense 與 CMake 編譯選項完全一致。

---

## 5. 如何打開並第一次執行

### 開啟 Workspace

* VS Code → **File** → **Open Workspace from File…** → 選 `your_project.code-workspace`。
  或命令列：`code your_project/your_project.code-workspace`。

### 第一次步驟

1. 狀態列 **CMake** 區域選擇 **Kit**：`CMake: Select a Kit`（GCC/Clang/MSVC）。
2. 若未自動 Configure：`CMake: Configure`。
3. **建置**：`Ctrl+Shift+B`（或按左側 CMake 面板的 Build）。
4. **執行**：`Terminal → Run Task… → Build and Run`（或用下方狀態列的 ▶）。
5. **除錯**：`Run and Debug`（選 `Debug (gdb/lldb via CMake)` 或 `Debug (MSVC / cppvsdbg)`）→ F5。

---

## 6. 常見錯誤與排除

| 症狀/訊息                                      | 可能原因                          | 解法                                                                             |
| ------------------------------------------ | ----------------------------- | ------------------------------------------------------------------------------ |
| `No kit selected` / `No compiler found`    | 尚未選擇 Kit 或未安裝編譯器              | `CMake: Select a Kit`、確認已安裝 GCC/Clang/MSVC，並把路徑加到 PATH                         |
| `program not found`（執行/除錯時）                | 可執行檔路徑錯誤                      | 改用 `${command:cmake.launchTargetPath}`；或調整 `CMAKE_RUNTIME_OUTPUT_DIRECTORY`    |
| `unable to start debugging`（Windows 下 gdb） | `miDebuggerPath` 路徑錯          | 把 `C:\\msys64\\mingw64\\bin\\gdb.exe` 換成你的實際安裝路徑                               |
| IntelliSense 找不到 headers                   | VS Code 與 CMake 設定不同          | 確認有 `C_Cpp.default.configurationProvider = ms-vscode.cmake-tools`，重新 Configure |
| `CMake configure` 失敗                       | 找不到依賴（OpenCV 等）或 generator 問題 | 安裝套件或加上 `-D` 參數；或在設定中指定 `"cmake.generator": "Ninja"` 並安裝 Ninja                 |
| macOS gdb 不能用                              | gdb 需要簽章；預設用 lldb             | 用 `lldb`（已在 launch.osx 指定），或自行簽章 gdb                                           |
| Windows 用 MSVC 但選到 gdb                     | 調試器不匹配                        | 選 `Debug (MSVC / cppvsdbg)` 這個 launch 設定                                       |
| WSL/路徑混亂                                   | Windows 與 WSL 檔案系統不同          | 在 WSL 裡開 WSL 的路徑；或用 VS Code Remote - WSL                                       |

---

## 7. 進階建議（可選）

* **固定 Generator 為 Ninja：**

  ```jsonc
  // settings
  "cmake.generator": "Ninja"
  ```
* **使用 CMake Presets（CMake 3.19+）**：在 `CMakePresets.json` 定義多組 configure/build 設定（Debug/Release、toolchain），CMake Tools 會自動讀取。
* **多根資料夾（multi-root）**：在 `"folders"` 加入其他路徑即可；Workspace 設定會套用在整個工作區。
* **產生 compile_commands.json**：CMake 預設可產生；如需保險，在 `CMakeLists.txt` 加：

  ```cmake
  set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
  ```

---

## 8. 我到底該在哪裡改「編譯選項」？

* **語言標準 / 最適化 / 巨集 / include 路徑** → 改 **CMakeLists.txt**（最穩定）：

  ```cmake
  set(CMAKE_CXX_STANDARD 14)
  target_compile_options(main PRIVATE -Wall -Wextra -O2)
  target_include_directories(main PRIVATE include)
  target_compile_definitions(main PRIVATE USE_FAST_PATH=1)
  ```
* **換編譯器** → `CMake: Select Kit`。
* **Debug/Release** → `CMake: Select Variant` 或 Preset。

> VS Code 的 `tasks/launch` 只是在呼叫 CMake「建置」與呼叫除錯器本身，不會決定真實的 `-I -D -std -O` 等旗標；這些由 **CMake** 與 **Kit** 決定。

---

## 9. 常見 QA

* **Q：不用 CMake 可以嗎？**
  A：可以，但專案一大會很痛苦。若要純 `tasks.json` 直接呼叫 `g++`，請自行維護所有來源檔與 include，容易失控。
* **Q：MinGW 與 MSVC 怎麼選？**
  A：若你的最終部署在 Windows，且依賴 Windows SDK/COM 等，MSVC 較佳；若追求與 Linux 相近的行為/ABI，MinGW 會更像 GCC/Clang。
* **Q：WSL 還是原生 Windows？**
  A：WSL 對 Linux 目標很方便；但 GUI/相機/OpenCV 視窗等在 WSL 要留意 X server 或 WSLg 支援。

---

## 10. 一鍵驗證流程（第一次克隆專案後）

1. `code your_project/your_project.code-workspace`
2. `CMake: Select a Kit` → 選你要的編譯器
3. `CMake: Configure`
4. `Ctrl+Shift+B`（Build）
5. `Run and Debug` → **Debug (gdb/lldb via CMake)**（或 **MSVC** 那個）
6. 看見「Hello, VS Code + CMake!」即成功

---

若你願意，把你現有的 `CMakeLists.txt`、實際使用的工具鏈與平台（Windows/MSYS2 or MSVC、macOS、Ubuntu）貼給我，我可以幫你把 Workspace、`launch` 與 `tasks` 做**最小侵入式**微調，確保按 F5 就能跑。
