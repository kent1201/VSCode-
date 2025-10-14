# VS Code 設定三件套：`settings.json` / `tasks.json` / `launch.json` 教學

## 0) 三者的責任分工（先有大圖）

* **`settings.json`**：編輯器與擴充套件「行為」的設定（User 或 Workspace 範圍）。例如 CMake build 目錄、Python 環境啟用行為、格式化規則。([code.visualstudio.com][1])
* **`tasks.json`**：定義「要跑的命令」（Build/Run/Lint/Script），可用 **Problem Matcher** 把輸出解析成錯誤/警告。([code.visualstudio.com][2])
* **`launch.json`**：定義「如何除錯」。可用變數替換（如 `${workspaceFolder}`、`${file}`）避免寫死路徑。([code.visualstudio.com][3])

---

## 1) 存放位置與作用範圍

* **User Settings（全域）** 與 **Workspace Settings（專案內）**；通常專案要可重現，建議把關鍵設定放 Workspace。可透過命令面板開啟 `Preferences: Open (User/Workspace) Settings (JSON)`。([code.visualstudio.com][1])
* 工作區（Workspace）模式下，檔案通常位於專案根的 **`.vscode/`** 目錄。([vscode-docs.readthedocs.io][4])

---

## 2) `settings.json`：常用配置（含 C++/Python）

```jsonc
{
  // --- 通用檔案風格 ---
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.trimFinalNewlines": true,

  // --- CMake / C++ ---
  "cmake.configureOnOpen": true,                        // 開專案就 configure
  "cmake.buildDirectory": "${workspaceFolder}/build",   // 統一 build 目錄
  // 讓 IntelliSense 直接吃 CMake 的 compile_commands.json
  "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",

  // --- Python / conda ---
  "python.terminal.activateEnvironment": true           // 終端自動啟動該環境
}
```

為什麼這樣設？

* `cmake.configureOnOpen` / `cmake.buildDirectory`：統一與自動化 CMake 的建置流程。([Stack Overflow][5])
* `C_Cpp.default.configurationProvider`：C/C++ IntelliSense 與實際編譯選項完全對齊。([Stack Overflow][5])
* `python.terminal.activateEnvironment`：開內建終端就啟用選取的 Python/conda 環境。([code.visualstudio.com][6])

---

## 3) `tasks.json`：建置/執行任務與 Problem Matcher

### 3.1 最小可用（以 CMake 建置 + 執行當前目標為例）

```jsonc
{
  "$schema": "vscode://schemas/tasks",
  "version": "2.0.0",
  "tasks": [
    {
      "label": "C++: Build (CMake)",
      "command": "cmake",
      "args": ["--build", "${command:cmake.buildDirectory}"],
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": "$gcc"   // 也可自訂 regex
    },
    {
      "label": "C++: Run Active Target",
      "type": "shell",
      "command": "${command:cmake.launchTargetPath}",
      "dependsOn": "C++: Build (CMake)"
    }
  ]
}
```

* **Problem Matcher** 會把任務輸出掃描成可點擊的錯誤列表；可用內建 `$gcc`、或自訂正則。([code.visualstudio.com][2])
* `tasks.json` **不可同時**有「全域的 `problemMatcher`」與「`tasks` 陣列」兩者並存（Schema 限制）。([code.visualstudio.com][7])
* `${command:cmake.launchTargetPath}` 由 CMake Tools 回傳目標可執行檔路徑；若為空多半代表尚未選目標或未成功 configure。([向量真值][8])

### 3.2 自訂 Problem Matcher（GCC/Clang 典型格式）

```jsonc
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build (custom matcher)",
      "command": "make",
      "args": [],
      "problemMatcher": {
        "owner": "cpp",
        "fileLocation": ["relative", "${workspaceFolder}"],
        "pattern": {
          "regexp": "^(.*):(\\d+):(\\d+):\\s+(warning|error):\\s+(.*)$",
          "file": 1, "line": 2, "column": 3, "severity": 4, "message": 5
        }
      }
    }
  ]
}
```

* 這種正則能把 `file:line:col: warning|error: message` 抓進 **Problems** 面板。([code.visualstudio.com][2])

### 3.3 Python 任務（以當下選取的解譯器執行目前檔案）

```jsonc
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Python: Run current file",
      "type": "shell",
      "command": "${command:python.interpreterPath}",
      "args": ["${file}"]
    }
  ]
}
```

* `python.interpreterPath` 與你在狀態列/命令面板選擇的環境相符。([code.visualstudio.com][6])

---

## 4) `launch.json`：除錯設定（C++/Python）

### 4.1 變數替換重點

* `${workspaceFolder}`：工作區根路徑；`${file}`：目前編輯檔；`${env:NAME}`：環境變數。可用於 `launch.json` 與 `tasks.json`。([code.visualstudio.com][9])
* multi-root 下變數「依所屬資料夾」解析；也能用 `:${folderName}` 指定。([code.visualstudio.com][10])

### 4.2 C++（gdb/lldb｜以 CMake 目標為主）

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C++: Debug active target",
      "type": "cppdbg",
      "request": "launch",
      "program": "${command:cmake.launchTargetPath}",
      "cwd": "${workspaceFolder}",
      "preLaunchTask": "C++: Build (CMake)",
      "stopAtEntry": false,
      "linux":   { "MIMode": "gdb",  "miDebuggerPath": "/usr/bin/gdb" },
      "osx":     { "MIMode": "lldb" },
      "windows": { "MIMode": "gdb",  "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe" }
    }
  ]
}
```

* C/C++ 除錯更多屬性（如 `coreDumpPath`/`dumpPath`）可參考官方文件。([code.visualstudio.com][11])
* 更一般性的除錯工作流與 `launch.json` 說明，見 VS Code Debug 指南。([code.visualstudio.com][3])

### 4.3 Python（以目前檔案做啟動程式）

```jsonc
{
  "version": "0.2.0",
  "configurations": [
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
```

* Python 調試的進階情境（Flask/Django、遠端、附加到行程）見官方 Python Debug 文檔。([code.visualstudio.com][6])

---

## 5) 變數與 Schema：好用參考

* **Variables Reference**（`${workspaceFolder}`、`${fileBasenameNoExtension}`、`${env:XXX}` 等清單）。([code.visualstudio.com][9])
* **Debug/launch 變數替換** 說明。([code.visualstudio.com][12])
* **Tasks 附錄（完整 Schema & 限制）**：例如不可同時定義全域 `problemMatcher` 與 `tasks`。([code.visualstudio.com][7])

---

## 6) Multi-root Workspace 注意事項

* 變數是「相對於該資料夾」解析；必要時用 `:${folderName}` 指名屬主。([code.visualstudio.com][10])
* 若出現 `${workspaceFolder} cannot be resolved`，通常是沒用「資料夾」或「Workspace」方式開啟（請先新增資料夾到 Workspace 再存成 `.code-workspace`）。([Stack Overflow][13])

---

## 7) 常見排錯對照表

| 症狀                                     | 可能原因                                   | 處理方式                                                                                               |
| -------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `No Kit Selected` / CMake 無法 Configure | 尚未選 CMake Kit/編譯器                      | `CMake: Select a Kit` 後再 Configure。([Stack Overflow][5])                                           |
| Debug 找不到可執行檔                          | `program` 寫死路徑且不符；或未成功 Configure/Build | 用 `${command:cmake.launchTargetPath}`；先 Build；若仍為 `null`，檢查目標/預設目標與 CMake Tools Issue。([向量真值][8])  |
| Problem Matcher 沒抓錯誤                   | 未設定或 regex 不符                          | 先試內建 `$gcc`；或自訂 regex；確認 `fileLocation` 與輸出格式。([code.visualstudio.com][2])                         |
| `${workspaceFolder}` 無法解析              | 不是用「開資料夾/Workspace」模式                  | 用 **Add Folder to Workspace**，或以 `.code-workspace` 開啟。([Stack Overflow][13])                       |
| Python 用錯環境                            | 未在狀態列選 Interpreter、或終端未啟用              | `Python: Select Interpreter`；啟用 `python.terminal.activateEnvironment`。([code.visualstudio.com][6]) |

---

## 8) 建議的最小專案範本（可直接拷貝到 `.vscode/`）

### `.vscode/settings.json`

```jsonc
{
  "cmake.configureOnOpen": true,
  "cmake.buildDirectory": "${workspaceFolder}/build",
  "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",
  "python.terminal.activateEnvironment": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.trimFinalNewlines": true
}
```

### `.vscode/tasks.json`

```jsonc
{
  "$schema": "vscode://schemas/tasks",
  "version": "2.0.0",
  "tasks": [
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
    {
      "label": "Python: Run current file",
      "type": "shell",
      "command": "${command:python.interpreterPath}",
      "args": ["${file}"]
    }
  ]
}
```

### `.vscode/launch.json`

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C++: Debug active target",
      "type": "cppdbg",
      "request": "launch",
      "program": "${command:cmake.launchTargetPath}",
      "cwd": "${workspaceFolder}",
      "preLaunchTask": "C++: Build (CMake)",
      "stopAtEntry": false,
      "linux":   { "MIMode": "gdb",  "miDebuggerPath": "/usr/bin/gdb" },
      "osx":     { "MIMode": "lldb" },
      "windows": { "MIMode": "gdb",  "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe" }
    },
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
```

---

## 9) 延伸閱讀（官方文件）

* User/Workspace Settings 與 Settings JSON。([code.visualstudio.com][1])
* Tasks 與 Problem Matchers（含完整 Schema 附錄）。([code.visualstudio.com][2])
* Debug/launch.json 與變數替換。([code.visualstudio.com][3])
* C/C++ 除錯（gdb/lldb/cppvsdbg）。([code.visualstudio.com][11])
* Python 調試。([code.visualstudio.com][6])
* Multi-root workspaces 變數解析原則。([code.visualstudio.com][10])

---

需要我把這三個檔 **合併進你的 `.code-workspace`**（讓 Workspace 單檔就帶著 tasks/launch/settings 的「集中管理版」）嗎？我可以幫你做一份最小侵入式、直接可用的範本。

[1]: https://code.visualstudio.com/docs/configure/settings?utm_source=chatgpt.com "User and workspace settings"
[2]: https://code.visualstudio.com/docs/debugtest/tasks?utm_source=chatgpt.com "Integrate with External Tools via Tasks"
[3]: https://code.visualstudio.com/docs/debugtest/debugging?utm_source=chatgpt.com "Debug code with Visual Studio Code"
[4]: https://vscode-docs.readthedocs.io/en/stable/customization/userandworkspace/?utm_source=chatgpt.com "User and Workspace Settings - vscode-docs"
[5]: https://stackoverflow.com/questions/62104648/defining-a-problemmatcher-in-vscode-tasks-schema-disagrees-with-docs?utm_source=chatgpt.com "Defining a ProblemMatcher in VSCode tasks -- schema ..."
[6]: https://code.visualstudio.com/docs/python/debugging?utm_source=chatgpt.com "Python debugging in VS Code"
[7]: https://code.visualstudio.com/docs/reference/tasks-appendix?utm_source=chatgpt.com "Appendix"
[8]: https://vector-of-bool.github.io/docs/vscode-cmake-tools/debugging.html?utm_source=chatgpt.com "Target Debugging and Launching — CMake Tools 1.4.0 ..."
[9]: https://code.visualstudio.com/docs/reference/variables-reference?utm_source=chatgpt.com "Variables reference"
[10]: https://code.visualstudio.com/docs/editing/workspaces/multi-root-workspaces?utm_source=chatgpt.com "Multi-root Workspaces"
[11]: https://code.visualstudio.com/docs/cpp/cpp-debug?utm_source=chatgpt.com "Debug C++ in Visual Studio Code"
[12]: https://code.visualstudio.com/docs/debugtest/debugging-configuration?utm_source=chatgpt.com "Visual Studio Code debug configuration"
[13]: https://stackoverflow.com/questions/61238046/workspacefolder-can-not-be-resolved-please-open-a-folder-on-visual-studi?utm_source=chatgpt.com "'${workspaceFolder}' can not be resolved. Please open a ..."
