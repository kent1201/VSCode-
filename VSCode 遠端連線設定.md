# VS Code 遠端開發安裝與設定指南（SSH / WSL / Dev Containers / Tunnels）

> 你可以把 VS Code 的 UI 跟計算環境分離：本機只跑介面，**真正的編譯/執行在遠端機器、WSL 或容器**。官方稱為 **Remote Development**。本文帶你從安裝到能連上遠端並完成第一個 Build/Debug。 ([code.visualstudio.com][1])

---

## 1) 安裝清單

1. **VS Code 本體**（Windows/macOS/Linux）。
2. **Remote Development 相關擴充：**

   * Remote - SSH
   * Dev Containers
   * WSL（僅 Windows 需要）
   * Remote - Tunnels（可選）
     可一次裝 **Remote Development** 擴充包。之後 VS Code 會在第一次連線時，把 **VS Code Server** 安到遠端環境裡。 ([code.visualstudio.com][1])

---

## 2) 你應該用哪一種？

| 情境                            | 推薦                    |
| ----------------------------- | --------------------- |
| 你有一台可 SSH 的 Linux/Server/EVK  | **Remote-SSH**        |
| 你在 **Windows** 又想用原生 Linux 工具 | **WSL + VS Code WSL** |
| 想要**乾淨、可重現**的工具鏈（含特定編譯器/SDK）  | **Dev Containers**    |
| 防火牆/無法開 SSH，但能出網路             | **Remote Tunnels**    |

各模式概覽與取捨，參考官方 Remote Development 說明。 ([code.visualstudio.com][1])

---

## 3) Remote-SSH：最常見的遠端連線

### 3.1 前置

* 遠端已啟動 **OpenSSH Server**（Linux 通常為 `sshd`）。
* 本機可 `ssh user@host` 直連或經跳板。 ([code.visualstudio.com][2])

### 3.2 安裝與首次連線

1. 在 VS Code 安裝 **Remote - SSH**。
2. 左下角綠色 **><** 圖示 → **Remote-SSH: Connect to Host…** → 輸入 `user@host`。
3. 第一次會自動在遠端安裝 VS Code Server，之後就像本機一樣編輯與除錯。 ([code.visualstudio.com][2])

### 3.3 SSH 設定（`~/.ssh/config`）

```ssh-config
Host my-board
  HostName 192.168.1.50
  User root
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 30
  ServerAliveCountMax 4
# 若有跳板機（ProxyJump）
Host my-target-via-jump
  HostName 10.0.0.20
  User dev
  ProxyJump jumpuser@jump.example.com
```

在 VS Code 連線清單選 `my-board` / `my-target-via-jump` 即可。官方 SSH 教學與逐步連線流程可參考。 ([code.visualstudio.com][2])

> 小提示：第一次連線後，**擴充套件裝在「遠端」或「本機」**會分開顯示。建議把語言相關（C/C++、Python…）裝在 *遠端* 以便偵測遠端工具鏈。 ([code.visualstudio.com][2])

---

## 4) WSL（Windows 專用）

若你在 Windows 上寫 Linux 程式（含 CMake、GCC、Python/conda），最乾淨的方式是裝 **WSL**，並在 **WSL內**打開 VS Code 視窗。步驟：

1. Windows 開啟 WSL 並安裝 Linux 發行版。
2. 安裝 VS Code 與 **WSL 擴充**。
3. 開 WSL 終端輸入 `code .`，VS Code 會自動在 WSL 佈署 Server，之後所有操作都在 WSL 內進行。 ([code.visualstudio.com][3])

---

## 5) Dev Containers（容器即開發機）

想要**團隊一致**的編譯環境或避免污染本機：

1. 安裝 Docker（或兼容執行器）。
2. VS Code 裝 **Dev Containers** 擴充。
3. 在專案加入 `.devcontainer/devcontainer.json`（或用命令 **Dev Containers: Add Dev Container Configuration Files…** 產生）。
4. 以 **Reopen in Container** 進入容器內開發、建置與除錯。 ([code.visualstudio.com][4])

---

## 6) Remote Tunnels（無 SSH 的替代方案）

在無法開 SSH/固定 IP 的環境，可用 **Remote - Tunnels** 建立安全通道，從任何地方連回那台機器的 VS Code Server（不需 SSH）。用法與限制見官方文件與 Marketplace。 ([code.visualstudio.com][5])

---

## 7) 首次開發：C++ 與 Python 範例

### 7.1 C++（以 Remote-SSH / WSL / Container 皆相同）

1. 在遠端安裝工具鏈：`gcc/g++` 或 `clang`、`cmake`、`gdb/lldb`。
2. 專案內放 `CMakeLists.txt`，在 VS Code 安裝 **CMake Tools**。
3. 狀態列 **CMake: Select Kit** → **Configure** → **Build** → F5 **Debug**。
   （`launch.json` 建議用 `${command:cmake.launchTargetPath}` 取得目標路徑） ([code.visualstudio.com][2])

### 7.2 Python（含 conda）

1. 在遠端安裝 `conda` 或 `venv`。
2. VS Code 選擇 **Python Interpreter** 指到遠端環境，啟用終端自動載入。
3. 直接 **Run** 或 **Debug current file** 即可。 ([code.visualstudio.com][3])

---

## 8) 效能與穩定小技巧

* **選對安裝位置**：語言擴充（C/C++、Python…）裝在「遠端」端。 ([code.visualstudio.com][2])
* **Keep-alive**：在 `~/.ssh/config` 增加 `ServerAliveInterval/CountMax`，降低閒置斷線。 ([code.visualstudio.com][2])
* **大檔與資料庫**：盡量讓 build cache、虛擬環境、`node_modules` 等留在遠端，避免檔案大量來回傳。 ([code.visualstudio.com][1])

---

## 9) 常見錯誤排除（FAQ）

| 症狀                                | 可能原因                 | 排除                                                                                               |
| --------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------ |
| **無法連 SSH** / 卡在安裝 VS Code Server | 防火牆、沒權限、家目錄空間不足      | 先本機 `ssh user@host` 驗證、檢查 `~/.vscode-server` 權限與磁碟，必要時指定安裝路徑或清空後重試。 ([code.visualstudio.com][2]) |
| 連上了但編譯器/套件抓不到                     | 擴充裝在本機不是遠端；或 PATH 不同 | 把擴充裝到「遠端」，在遠端 Terminal 檢查 `which g++/python`。 ([code.visualstudio.com][2])                       |
| Windows 想用 Linux 工具               | 忘了用 WSL 視窗           | 請用 **WSL: Reopen Folder in WSL** 或在 WSL 終端 `code .` 開啟。 ([code.visualstudio.com][3])             |
| 想要一致環境、避免「我這能跑你那不能」               | 本機與遠端/同事環境不同         | 改用 **Dev Containers**；用 `devcontainer.json` 鎖定工具鏈版本。 ([code.visualstudio.com][4])                |
| 沒法開 SSH，但要遠端編輯那台機器                | 網路限制                 | 用 **Remote Tunnels**；不需要 SSH。 ([code.visualstudio.com][5])                                       |

更多 Remote Development 常見 Q&A 見官方 FAQ。 ([code.visualstudio.com][6])

---

## 10) 進一步學習

* Remote-SSH 教學與概念、端到端流程。 ([code.visualstudio.com][2])
* WSL 開發完整教學（Windows 官方 Learn）。 ([Microsoft Learn][7])
* Dev Containers 教學與設定。 ([code.visualstudio.com][8])
* Remote Tunnels 說明。 ([code.visualstudio.com][5])
* 官方 Remote Development 課程路徑。 ([Microsoft Learn][9])

---

### 附錄：建議的 `.vscode` 最小組態（遠端/WSL/容器通用）

```jsonc
// .vscode/settings.json
{
  "cmake.configureOnOpen": true,
  "cmake.buildDirectory": "${workspaceFolder}/build",
  "C_Cpp.default.configurationProvider": "ms-vscode.cmake-tools",
  "python.terminal.activateEnvironment": true
}
```

```jsonc
// .vscode/tasks.json
{
  "$schema": "vscode://schemas/tasks",
  "version": "2.0.0",
  "tasks": [
    { "label": "C++: Build", "command": "cmake",
      "args": ["--build", "${command:cmake.buildDirectory}"],
      "group": { "kind": "build", "isDefault": true }, "problemMatcher": "$gcc" },
    { "label": "Run Active Target", "type": "shell",
      "command": "${command:cmake.launchTargetPath}", "dependsOn": "C++: Build" }
  ]
}
```

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    { "name": "C++: Debug active target", "type": "cppdbg", "request": "launch",
      "program": "${command:cmake.launchTargetPath}", "cwd": "${workspaceFolder}",
      "preLaunchTask": "C++: Build",
      "linux": { "MIMode": "gdb", "miDebuggerPath": "/usr/bin/gdb" },
      "osx":   { "MIMode": "lldb" },
      "windows": { "MIMode": "gdb", "miDebuggerPath": "C:\\\\msys64\\\\mingw64\\\\bin\\\\gdb.exe" } }
  ]
}
```

---

[1]: https://code.visualstudio.com/docs/remote/remote-overview?utm_source=chatgpt.com "VS Code Remote Development"
[2]: https://code.visualstudio.com/docs/remote/ssh?utm_source=chatgpt.com "Remote Development using SSH"
[3]: https://code.visualstudio.com/docs/remote/wsl?utm_source=chatgpt.com "Developing in WSL"
[4]: https://code.visualstudio.com/docs/devcontainers/containers?utm_source=chatgpt.com "Developing inside a Container"
[5]: https://code.visualstudio.com/docs/remote/tunnels?utm_source=chatgpt.com "Developing with Remote Tunnels"
[6]: https://code.visualstudio.com/docs/remote/faq?utm_source=chatgpt.com "Remote Development FAQ"
[7]: https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode?utm_source=chatgpt.com "Get started using VS Code with WSL"
[8]: https://code.visualstudio.com/docs/devcontainers/tutorial?utm_source=chatgpt.com "Dev Containers tutorial"
[9]: https://learn.microsoft.com/en-us/training/paths/remote-development-vs-code/?utm_source=chatgpt.com "Remote development with Visual Studio Code - Training"
