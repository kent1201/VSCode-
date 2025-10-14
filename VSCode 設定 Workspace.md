# VS Code 通用 Workspace 教學（含 C++/CMake 與 Python/Conda）

> 目標
>
> * 從「什麼是 Workspace」開始，建立可重複使用的一份 `.code-workspace`。
> * 同時支援 **C++（CMake Tools）** 與 **Python（含 Miniconda/Conda）**。
> * 清楚知道該把設定放哪裡（`settings.json` / `tasks.json` / `launch.json`）。
> * 提供常見錯誤的排除方法。

---

## 0) 什麼是 Workspace？單根 vs. 多根（multi-root）

* **Workspace** 是 VS Code 打開的一組資料夾集合；可以是一個資料夾，也可以是多個資料夾（multi-root）。多根適合 mono-repo 或同時開多個相依專案。Workspace 設定可存成一個 **`.code-workspace`** 檔，方便版本控管與分享。([code.visualstudio.com][1])

---

## 1) 先安裝的擴充套件（Extensions）

* **C++/CMake：** `ms-vscode.cpptools`、`ms-vscode.cmake-tools`。官方 CMake Tools 教學與「Configure/Build/Debug」操作詳見 VS Code 文檔。([code.visualstudio.com][2])
* **Python：** `ms-python.python`（自動偵測 interpreters/conda envs）、（可選）`ms-python.vscode-python-envs` 管理環境。([code.visualstudio.com][3])

---

## 2) 建議的 Workspace 檔案骨架

> 建議將語言共用設定放在 `.code-workspace` 的 `"settings"`，並在 `"tasks"`、`"launch"` 放常用工作流程。

```jsonc
{
  "folders": [
    { "path": "." },
    // 需要 multi-root 時可再加：
    // { "path": "cpp" },
    // { "path": "python" }
  ],

  "extensions": {
    "recommendations": [
      "ms-vscode.cpptools",
      "ms-vscode.cmake-tools",
      "ms-python.python",
      "ms-python.vscode-python-envs"
    ]
  },

  "settings": {
    // 通用檔案風格
    "files.trimTrailingWhitespace": true,
    "files.insertFinalNewline": true,
    "files.trimFinalNewlines": true,

    // === C++ / CMake Tools ===
    "cmake.configureOnOpen": true,                     // 打開就自動 Configure
    "cmake.buildDirectory": "${workspaceFolder}/build",// 固定 build 目錄
    // 讓 C/C++ IntelliSense 直接吃 CMake 產生的 compile_commands.json
    "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",

    // === Python ===
    // 讓每個資料夾可選不同解譯器 / conda 環境（搭配狀態列選擇）
    "python.terminal.activateEnvironment": true
  },

  "tasks": {
    "version": "2.0.0",
    "tasks": [
      // --- C++: 以 CMake Tools 為主 ---
      {
        "label": "C++: Build (CMake)",
        "command": "cmake",
        "args": ["--build", "${command:cmake.buildDirectory}"],
        "group": { "kind": "build", "isDefault": true },
        "problemMatcher": "$gcc"
      },
      {
        "label": "C++: Run Active Target",
        "type": "shell",
        "command": "${command:cmake.launchTargetPath}",
        "dependsOn": "C++: Build (CMake)"
      },

      // --- Python: 以目前選擇的 Interpreter 執行 ---
      {
        "label": "Python: Run current file",
        "type": "shell",
        "command": "${command:python.interpreterPath}",
        "args": ["${file}"],
        "problemMatcher": []
      }
    ]
  },

  "launch": {
    "version": "0.2.0",
    "configurations": [
      // --- C++ 調試（gdb/lldb）---
      {
        "name": "C++: Debug active target",
        "type": "cppdbg",
        "request": "launch",
        "program": "${command:cmake.launchTargetPath}",
        "cwd": "${workspaceFolder}",
        "preLaunchTask": "C++: Build (CMake)",
        "stopAtEntry": false,
        "linux":  { "MIMode": "gdb",  "miDebuggerPath": "/usr/bin/gdb" },
        "osx":    { "MIMode": "lldb" },
        "windows":{ "MIMode": "gdb",  "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe" }
      },

      // --- Python 調試 ---
      {
        "name": "Python: Debug current file",
        "type": "python",
        "request": "launch",
        "program": "${file}",
        "console": "integratedTerminal",
        "justMyCode": true
      }
    ]
  }
}
```

**為什麼這樣設計？**

* `cmake.configureOnOpen` + `cmake.buildDirectory`：統一 CMake 的輸出位置，進出 repo 都可預期。([code.visualstudio.com][2])
* `C_Cpp.default.configurationProvider`：IntelliSense 與實際 CMake 編譯參數一致，避免 include/defines 走鐘。([code.visualstudio.com][2])
* `${command:cmake.launchTargetPath}`：可執行檔路徑由 CMake Tools 提供，無論單/多組態或平台都穩。([code.visualstudio.com][2])
* Python 端以 **狀態列選擇的 Interpreter/conda env** 為準，任務與除錯都跟著跑。([code.visualstudio.com][3])

---

## 3) C++（CMake Tools）最小流程

1. **安裝編譯器/除錯器**（GCC/Clang + gdb 或 MSVC + cppvsdbg）。
2. 打開 Workspace 後，狀態列 **CMake** 區塊 → `Select Kit` 選工具鏈 → 自動/手動 **Configure**。
3. `Ctrl+Shift+B` 或執行任務 **C++: Build (CMake)**。
4. F5 使用 **C++: Debug active target**。
   （底層即是 VS Code 透過 CMake Tools 的「Configure/Build/Launch」工作流。）([code.visualstudio.com][2])

> 小技巧：若要產生 Hello World 專案，CMake Tools 有 Quick Start 向導。([code.visualstudio.com][4])

---

## 4) Python（含 Miniconda/Conda）最小流程

1. 安裝 **Miniconda/Anaconda**，建立環境：

   ```bash
   conda create -n myenv python=3.11 -y
   conda activate myenv
   ```
2. **從啟動了環境的 shell 開 VS Code**（確保 PATH/環境變數正確），或在 VS Code 狀態列選 **Python Interpreter** 指向該環境。([code.visualstudio.com][5])
3. 建議啟用「在整合終端啟動時自動啟用環境」：本文範例已在 `python.terminal.activateEnvironment: true`。([code.visualstudio.com][3])
4. 執行檔案：任務 **Python: Run current file** 或用 VS Code 內建的「Run Python File」/ Debugger。([code.visualstudio.com][3])

---

## 5) tasks / launch / settings 放哪裡？

* **Workspace 層級**（建議）：「團隊共用的預設」放 `.code-workspace` 或專案根的 `.vscode/`。`tasks.json` 只在 Workspace（資料夾）情境可用。([code.visualstudio.com][6])
* **User 層級**（個人）：個人偏好放 **User Settings**。VS Code 會合併（User < Workspace < Folder）。([code.visualstudio.com][7])

想查 `tasks.json` 的完整 schema 與可用變數，可參考官方附錄與變數替換文件：([code.visualstudio.com][8])

---

## 6) Multi-root Workspace（單檔統一管理多資料夾）

* 把多個專案資料夾加進 `"folders"` 陣列；各資料夾可以有各自的 `.vscode` 設定，Workspace 也能提供全域設定與除錯組態。特別適合 mono-repo 或是同時開 C++/Python 子專案。([code.visualstudio.com][9])
* 實務上常見做法：同一個 `.code-workspace` 檔管理多個 package/service，每個資料夾各自選擇 Python/conda 環境或 CMake Kit。([Microsoft for Developers][10])

---

## 7) 疑難排解（FAQ）

| 症狀                                    | 可能原因                                          | 處理方式                                                                                                          |
| ------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| CMake Tools 狀態列顯示 **No Kit Selected** | 未選擇編譯器                                        | `CMake: Select a Kit` 選 GCC/Clang/MSVC；再 `CMake: Configure`。([vector-of-bool.github.io][11])                  |
| `Build` 成功但 Debug 找不到執行檔              | `program` 指錯路徑                                | 在 `launch`/`tasks` 使用 `${command:cmake.launchTargetPath}` 由 CMake Tools 提供。([code.visualstudio.com][2])       |
| Python/conda 環境沒被偵測到                  | 尚未在該資料夾建立 `.py` 或未刷新                          | Command Palette：**Python: Select Interpreter**；必要時先開一個 `.py` 再重整。([code.visualstudio.com][3])                 |
| `tasks.json` 欄位不確定                    | 想查 schema/變數                                  | 參考官方 tasks 文件與變數替換、或在 JSON 中使用 IntelliSense。([code.visualstudio.com][8])                                      |
| Multi-root 除錯看不到組態                    | 組態沒放在 Workspace 層或沒保存為 `.code-workspace`      | 將通用組態放在 `launch` 的 workspace 區塊；以「Open Workspace from File…」開啟 `.code-workspace`。([code.visualstudio.com][9]) |
| 從 conda 啟動後 VS Code 裡不是同一個環境          | 沒用啟動了環境的 shell 開 VS Code，或未在狀態列選對 Interpreter | 在 **已 activate 的終端**中執行 `code .`；或用狀態列/命令面板切換 Interpreter。([code.visualstudio.com][5])                        |

---

## 8) 進階：JSON Schema 與自動補全

* VS Code 內建 JSON 語言服務，支援 schema 與提示；在 `tasks.json` / `launch.json` 可直接用 IntelliSense 補完。([code.visualstudio.com][12])
* 也可在檔案頂端指定對應 schema，例如：

  ```jsonc
  { "$schema": "vscode://schemas/tasks", "version": "2.0.0", "tasks": [] }
  ```

  （屬內部 schema URI，VS Code 能識別並提供提示。）([Stack Overflow][13])

---

## 9) 一次檢查清單（初次到新機/新專案）

1. 安裝 VS Code + 必要 extensions（C++/CMake、Python）。([code.visualstudio.com][2])
2. 安裝編譯器/除錯器（GCC/Clang/gdb 或 MSVC/cppvsdbg）。([code.visualstudio.com][2])
3. 安裝 Miniconda/Anaconda（如需）並建立環境。([Anaconda][14])
4. 以 **`.code-workspace`** 開啟專案（非單純開資料夾）。([code.visualstudio.com][1])
5. C++：`Select Kit` → `Configure` → `Build` → `Debug`。([code.visualstudio.com][2])
6. Python：狀態列選擇 Interpreter / conda env → Run/Debug。([code.visualstudio.com][3])

---

### 結語

* **Workspace** 讓不同語言、不同專案在同一視窗共存，並把共用設定版本化。
* **C++ 交給 CMake Tools**：Kit/Configure/Build/Launch 一條龍；`launchTargetPath` 讓路徑免煩惱。([code.visualstudio.com][2])
* **Python 交給 Python 擴充與 conda**：用狀態列切 Interpreter，終端自動啟用環境。([code.visualstudio.com][3])


[1]: https://code.visualstudio.com/docs/editing/workspaces/workspaces?utm_source=chatgpt.com "What is a VS Code workspace?"
[2]: https://code.visualstudio.com/docs/cpp/cmake-linux?utm_source=chatgpt.com "Get started with CMake Tools on Linux"
[3]: https://code.visualstudio.com/docs/languages/python?utm_source=chatgpt.com "Python in Visual Studio Code"
[4]: https://code.visualstudio.com/docs/cpp/cmake-quickstart?utm_source=chatgpt.com "Create a CMake hello world project with CMake Quick Start"
[5]: https://code.visualstudio.com/docs/python/environments?utm_source=chatgpt.com "Python environments in VS Code"
[6]: https://code.visualstudio.com/docs/debugtest/tasks?utm_source=chatgpt.com "Integrate with External Tools via Tasks"
[7]: https://code.visualstudio.com/docs/configure/settings?utm_source=chatgpt.com "User and workspace settings"
[8]: https://code.visualstudio.com/docs/reference/tasks-appendix?utm_source=chatgpt.com "Appendix"
[9]: https://code.visualstudio.com/docs/editing/workspaces/multi-root-workspaces?utm_source=chatgpt.com "Multi-root Workspaces"
[10]: https://devblogs.microsoft.com/ise/multi_root_workspaces_in_visual_studio_code/?utm_source=chatgpt.com "Multi Root Workspaces in Visual Studio Code"
[11]: https://vector-of-bool.github.io/docs/vscode-cmake-tools/kits.html?utm_source=chatgpt.com "CMake Kits — CMake Tools 1.4.0 documentation"
[12]: https://code.visualstudio.com/docs/languages/json?utm_source=chatgpt.com "Editing JSON with Visual Studio Code"
[13]: https://stackoverflow.com/questions/73333047/where-can-i-find-schema-files-for-vs-codes-json-files-like-launch-json-and-task?utm_source=chatgpt.com "Where can I find schema files for VS Code's JSON files like ..."
[14]: https://www.anaconda.com/docs/getting-started/working-with-conda/ide-tutorials/vscode?utm_source=chatgpt.com "Visual Studio Code (VS Code) - conda environment"
