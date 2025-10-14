# VS Code Extensions 安裝與設定（含 settings.json 範例）

## 1) 安裝與管理 Extensions（基礎操作）

* 開啟擴充檢視：**Ctrl+Shift+X** → 搜尋名稱 → **Install**。也可從 Marketplace 頁面安裝。啟用 Settings Sync 可在多台機器同步擴充；必要時可選「**Install (Do not Sync)**」或指定**安裝舊版本**（右鍵擴充 → *Install Another Version*）。 ([code.visualstudio.com][1])
* Extension 的**自訂設定**可在 **Settings** 的「Extensions」區塊找到；也能在擴充頁面的 **Feature Contributions** 檢視該擴充提供的設定/指令/語法支援。 ([code.visualstudio.com][2])

> 變數替換（tasks/launch/settings 可用）：例如 `${workspaceFolder}`、`${config:editor.fontSize}`。清單見官方 Variables 參考。 ([code.visualstudio.com][3])

---

## 2) 推薦安裝清單（通用開發）

* **C/C++**：`ms-vscode.cpptools`（IntelliSense/除錯） ([code.visualstudio.com][4])
* **CMake Tools**：`ms-vscode.cmake-tools`（Configure/Build/Debug 一條龍） ([code.visualstudio.com][5])
* **Python**：`ms-python.python`（解譯器/環境/調試/測試整合） ([code.visualstudio.com][6])
* **Prettier**（前端/文件格式化） ([DigitalOcean][7])
* **ESLint**（JS/TS Lint & Fix，含 Flat Config 支援） ([marketplace.visualstudio.com][8])
* **Docker / Dev Containers**（容器內開發） ([code.visualstudio.com][9])

> 提醒：近月有**惡意擴充**事件，請優先安裝**官方/高信任來源**並檢查權限；發現異常請停用/移除。 ([TechRadar][10])

---

## 3) 在 `settings.json` 調整 Extension 參數（通用示例）

> 開啟方式：Command Palette → **Preferences: Open Settings (JSON)**。

```jsonc
{
  // —— 檔案風格 ——
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.trimFinalNewlines": true,

  // —— C/C++ & CMake Tools —— 
  // 讓 IntelliSense 直接吃 CMake 產生的 compile_commands.json
  "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",   // :contentReference[oaicite:10]{index=10}
  "cmake.configureOnOpen": true,                                     // 開資料夾自動 Configure  :contentReference[oaicite:11]{index=11}
  "cmake.buildDirectory": "${workspaceFolder}/build",
  // 若系統 CMake 不在 PATH，可指定：
  // "cmake.cmakePath": "/usr/local/bin/cmake",                      // :contentReference[oaicite:12]{index=12}

  // —— Python —— 
  "python.terminal.activateEnvironment": true,                        // 終端自動啟用選取的環境  :contentReference[oaicite:13]{index=13}

  // —— Prettier —— 
  "editor.defaultFormatter": "esbenp.prettier-vscode",                // 設為預設格式化器  :contentReference[oaicite:14]{index=14}
  "editor.formatOnSave": true,

  // —— ESLint —— 
  "eslint.useFlatConfig": true,                                       // 依 ESLint 版本自動切換支援  :contentReference[oaicite:15]{index=15}
  "eslint.alwaysShowStatus": true,
  "editor.codeActionsOnSave": { "source.fixAll.eslint": "explicit" }  // 儲存時以 ESLint 修正

  // —— Docker / Containers —— 
  // 多為 UI 操作；如需容器開發請在 .devcontainer/ 設定（見官方教學）  :contentReference[oaicite:16]{index=16}
}
```

---

## 4) 各語言/工具的「最小可用」設定與實戰

### 4.1 C/C++ + CMake Tools（建議組合）

**安裝**：`C/C++`、`CMake Tools`。
**關鍵設定**：把 IntelliSense provider 指向 CMake Tools；固定 build 目錄，必要時指定 `cmake.cmakePath`。 ([code.visualstudio.com][11])
**首次流程**：Select Kit → Configure → Build → Debug（CMake Tools 教學）。 ([code.visualstudio.com][5])

### 4.2 Python + Conda/venv

**安裝**：`Python`。
**流程**：在狀態列選 **Python Interpreter**（如 conda env）。新增/變更 conda 環境時可用 **刷新圖示**重載清單。啟用 `python.terminal.activateEnvironment` 讓整合終端自動 `activate`。 ([code.visualstudio.com][6])

### 4.3 Prettier（統一代碼風格）

**安裝**：`esbenp.prettier-vscode`。
**設定**：將 `editor.defaultFormatter` 設為 Prettier，並開啟 `formatOnSave`。 ([DigitalOcean][7])

### 4.4 ESLint（JS/TS）

**安裝**：`dbaeumer.vscode-eslint`。
**設定**：依 ESLint 版本啟用 `eslint.useFlatConfig`；儲存時自動 `fixAll`。 ([marketplace.visualstudio.com][8])

### 4.5 Docker / Dev Containers（容器內開發）

**安裝**：`Dev Containers`、`Docker`。
**用法**：加入 `.devcontainer/devcontainer.json`，選 **Reopen in Container**；容器工具視圖與管理見官方文件。 ([code.visualstudio.com][9])

---

## 5) Extension 設定的「查找與覆寫」技巧

1. **在哪裡看這個擴充能設定什麼？**

* Settings → **Extensions** 區塊（搜尋該擴充），或在擴充頁的 **Feature Contributions** 標籤。 ([code.visualstudio.com][2])

2. **想知道所有可用設定的預設值？**

* 開啟 **Default Settings**（Command Palette 搜尋 *defaultSettings*）以瀏覽全部可設定項目（唯讀）。 ([Stack Overflow][12])

3. **在 tasks/launch 中讀取現有設定**

* 使用變數 `${config:some.setting}`（例如引用 `editor.fontSize`）。 ([code.visualstudio.com][3])

---

## 6) 工作區（Workspace） vs 使用者（User）設定

* 建議把**專案需要的一致設定**放在 **Workspace**（專案 `.vscode/settings.json` 或 `.code-workspace`），把個人偏好放 **User**，降低團隊環境差異。官方設定層級與合併規則見文件。 ([code.visualstudio.com][2])

---

## 7) 常見問題（FAQ / Troubleshooting）

| 症狀                          | 可能原因                       | 解法                                                                                                 |
| --------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------- |
| 安裝後功能沒生效                    | 擴充裝在**本機**但你在**遠端/WSL**開發  | 把擴充安裝到**遠端**環境（左下角遠端狀態列會提示）。 ([code.visualstudio.com][3])                                          |
| CMake Tools 找不到 CMake       | CMake 未在 PATH              | 設定 `"cmake.cmakePath"` 指向 CMake 可執行檔。 ([marketplace.visualstudio.com][13])                         |
| C/C++ IntelliSense 與實際編譯不一致 | 未使用 CMake Tools 當 provider | 設定 `"C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools"`。 ([code.visualstudio.com][11]) |
| Python 找不到新建的 conda 環境      | 清單未刷新                      | 在 **Select Interpreter** 視窗按**刷新**；或重啟 VS Code。 ([code.visualstudio.com][6])                       |
| 想強制用 Prettier               | 沒設預設 formatter             | 設 `"editor.defaultFormatter": "esbenp.prettier-vscode"` 並開啟 `formatOnSave`。 ([DigitalOcean][7])    |
| 惡意擴充風險                      | 市集曾出現惡意套件                  | 優先選官方/高評分；審核權限；關閉自動更新；留意公告。 ([TechRadar][10])                                                      |

---

## 8) 一鍵檢查清單（新專案/新機器）

1. 安裝 VS Code 與必要擴充（C/C++、CMake Tools、Python、Prettier、ESLint、Docker/Containers）。 ([code.visualstudio.com][5])
2. 在專案 `.vscode/settings.json` 放入上方「**通用示例**」設定（可依專案調整）。
3. C/C++ 專案：**Select Kit → Configure → Build → Debug**（CMake Tools 流程）。 ([code.visualstudio.com][5])
4. Python 專案：**Select Interpreter** → 確認終端會自動 `activate`。 ([code.visualstudio.com][6])
5. 前端專案：設定 **Prettier** 為預設 formatter、ESLint `fixAll` on save。 ([DigitalOcean][7])

---

[1]: https://code.visualstudio.com/docs/getstarted/extensions?utm_source=chatgpt.com "Use extensions in Visual Studio Code"
[2]: https://code.visualstudio.com/docs/configure/settings?utm_source=chatgpt.com "User and workspace settings"
[3]: https://code.visualstudio.com/docs/reference/variables-reference?utm_source=chatgpt.com "Variables reference"
[4]: https://code.visualstudio.com/docs/cpp/customize-cpp-settings?utm_source=chatgpt.com "C++ extension settings reference"
[5]: https://code.visualstudio.com/docs/cpp/cmake-linux?utm_source=chatgpt.com "Get started with CMake Tools on Linux"
[6]: https://code.visualstudio.com/docs/python/environments?utm_source=chatgpt.com "Python environments in VS Code"
[7]: https://www.digitalocean.com/community/tutorials/how-to-format-code-with-prettier-in-visual-studio-code?utm_source=chatgpt.com "Format Code with Prettier in Visual Studio Code: Setup Guide"
[8]: https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint&utm_source=chatgpt.com "VS Code ESLint extension"
[9]: https://code.visualstudio.com/docs/containers/overview?utm_source=chatgpt.com "Containers in Visual Studio Code"
[10]: https://www.techradar.com/pro/security/vscode-market-struck-by-huge-influx-of-malicious-whitecobra-extensions-so-be-warned?utm_source=chatgpt.com "VSCode market struck by huge influx of malicious WhiteCobra extensions - so be warned"
[11]: https://code.visualstudio.com/docs/cpp/configure-intellisense?utm_source=chatgpt.com "Configure C/C++ IntelliSense"
[12]: https://stackoverflow.com/questions/71633612/where-to-find-documentation-on-all-available-vs-code-editor-settings-json-settin?utm_source=chatgpt.com "Where to find documentation on all available VS code ..."
[13]: https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools&utm_source=chatgpt.com "CMake Tools"
