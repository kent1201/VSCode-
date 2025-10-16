# 你到底在做什麼：從原始碼到執行檔的 4 個階段

當你敲一行編譯指令時，其實在背後會跑「**預處理 → 編譯 → 彙編 → 連結**」四階段：

1. **預處理**：處理 `#include`、`#define` 等巨集。
2. **編譯**：把 C++ 原始碼變成組語。
3. **彙編**：把組語變成 **物件檔**（`.o/.obj`）。
4. **連結**：把很多個物件檔和函式庫黏在一起，做出 **執行檔**。
   這是 GCC 官方描述的標準流程與常見副檔名。([gcc.gnu.org][1])

> 物件檔（object file）是尚未連結的機器碼，通常不能直接執行；它會在連結階段被合併成可執行檔或函式庫。([維基百科][2])

---

# 我需要哪個編譯器？

* **Linux（常見）**：`g++`（GCC 的 C++ 前端）。
* **macOS**：`clang++`（Xcode Command Line Tools 內含）。若尚未安裝，可在終端機執行 `xcode-select --install` 安裝工具。([clang.llvm.org][3])
* **Windows**：建議用 **MSVC**（Visual C++ 編譯器 `cl`）。先開 **Developer Command Prompt / PowerShell** 再執行指令（這個殼會幫你設定好環境變數）。([Microsoft Learn][4])

  * 想在 Windows 用 Linux 指令列體驗：可裝 **WSL**：`wsl --install`。([Microsoft Learn][5])

---

# 第一支程式：一步做完（單檔）

建立 `main.cpp`：

```cpp
#include <iostream>
int main() {
  std::cout << "Hello C++\n";
}
```

## Linux / macOS

```bash
# 用 C++14、開警告、加除錯資訊(-g)、不要最佳化(-O0)、輸出到 hello
g++ -std=c++14 -Wall -Wextra -O0 -g main.cpp -o hello
./hello
```

* `-std=c++14`：選 C++ 標準版本。GCC/Clang 都支援用 `-std=c++17`、`-std=c++20` 等指定。([clang.llvm.org][3])
* `-Wall -Wextra`：開啟常見與額外警告。這是 GCC/Clang 的推薦做法。([gcc.gnu.org][6])
* `-g`：產生除錯資訊，讓 gdb/lldb 能對應原始碼。([gcc.gnu.org][6])
* `-O0`/`-O2`/`-O3`：最佳化等級（開發或除錯常用 `-O0` 或 `-O1`；釋出用 `-O2`/`-O3`）。([gcc.gnu.org][6])

> **為什麼用 `g++` 而不是 `gcc`？** 因為 `g++` 會自動幫你正確連到 C++ 標準函式庫；GCC 文件也建議編 C++ 時用 `g++`。([gcc.gnu.org][7])

## Windows（MSVC）

在 **Developer Command Prompt** 中：

```bat
cl /std:c++14 /W4 /Zi main.cpp
main.exe
```

* `/std:c++14`：指定 C++ 標準（也可用 `/std:c++17`、`/std:c++20`）。([Microsoft Learn][8])
* `/W4`：較高的警告等級（比預設更嚴謹，/Wall 太吵通常不建議）。([Microsoft Learn][9])
* `/Zi` 或 `/Z7`：產生除錯資訊（`/Zi` 產生 PDB，常見）。([Microsoft Learn][10])

---

# 拆步驟來看：先編成物件檔，再連結

把流程拆開比較好理解，也利於多檔案專案：

**Linux / macOS（GCC/Clang 同理）**

```bash
# 只編譯（不連結）：-c 會輸出物件檔 .o
g++ -std=c++14 -Wall -Wextra -O2 -c foo.cpp -o foo.o
g++ -std=c++14 -Wall -Wextra -O2 -c main.cpp -o main.o

# 連結：把多個 .o 黏在一起成執行檔
g++ foo.o main.o -o app
```

* `-c` 代表停在「產生物件檔」這一步，之後再連結。這個分段做法就是把四階段拆開。([gcc.gnu.org][1])

**Windows（MSVC）**

```bat
cl /std:c++14 /W4 /O2 /c foo.cpp
cl /std:c++14 /W4 /O2 /c main.cpp
link foo.obj main.obj /OUT:app.exe
```

（用 `cl` 也可一口氣完成，這裡示範分步驟。）

---

# `#include` 怎麼找到標頭？`-I` 是什麼？

* `#include "xxx.h"`：先從**目前檔案所在資料夾**找，再找系統路徑。
* `#include <xxx>`：只在**系統預設路徑**找。
* 你可以用 `-I path` 把自家 `include/` 加進搜尋路徑。這些行為是 C/C++ 預處理器的既定規則。([gcc.gnu.org][11])
* 微軟的文件也說明 `#include` 與標頭檔扮演的角色。([Microsoft Learn][12])

---

# 連結外部函式庫：`-L`、`-l` 與順序

以 Linux/macOS 為例（GCC）：

```bash
# 假設有 libmylib.a 或 libmylib.so 在 /opt/mylib/lib
g++ main.o -L/opt/mylib/lib -lmylib -o app
```

* `-L<dir>`：告訴連結器去哪裡找庫。
* `-l<name>`：連結名為 `lib<name>.a`（靜態）或 `lib<name>.so`（共享）的庫。
* **順序很重要**：通常把要用的物件檔放前面、函式庫放後面。以上都是 GCC「Link Options」的規則。([gcc.gnu.org][13])

> 小撇步：很多第三方庫支援 `pkg-config`，可自動吐出需要的 `-I/-L/-l` 參數：
> `g++ main.cpp $(pkg-config --cflags --libs sdl2)` 。`pkg-config` 的用途就是提供**編譯與連結**所需旗標。([people.freedesktop.org][14])

---

# 做自己的函式庫（靜態 / 共享）

## 靜態庫（`.a` / Windows `.lib`）

```bash
# Linux/macOS：先做物件檔，再打包成靜態庫
g++ -c add.cpp -o add.o
ar rcs libcalc.a add.o     # 建立 libcalc.a
g++ main.cpp -L. -lcalc -o app
```

Red Hat 的教學示範了用 `ar rcs` 建靜態庫與連結的方法。([docs.redhat.com][15])

**MSVC**（靜態庫用 `lib.exe` 產生）：

```bat
cl /c add.cpp
lib /OUT:calc.lib add.obj
cl main.cpp calc.lib
```

`lib.exe` 是微軟的 Library Manager。([Microsoft Learn][16])

## 共享庫（`.so` / macOS `.dylib` / Windows `.dll`）

```bash
# Linux/macOS：-fPIC 產可重定位碼，-shared 產生共享庫
g++ -fPIC -c add.cpp -o add.o
g++ -shared add.o -o libcalc.so
g++ main.cpp -L. -lcalc -o app
```

建立共享庫與其連結方式，參考 GCC 連結選項說明。([gcc.gnu.org][13])

---

# CMake 只要這樣開始就好（最小可用）

雖然這篇聚焦「編譯原理與指令」，但你之後九成會交給 CMake 管。最迷你的 CMakeLists：

```cmake
cmake_minimum_required(VERSION 3.16)
project(HelloCMake CXX)
add_executable(hello src/main.cpp)
```

用法（跨平台通吃）：

```bash
cmake -S . -B build
cmake --build build
```

`add_executable()` 與 `target_link_libraries()` 是 CMake 連結與產生執行檔的核心 API。官方文件在這裡。([cmake.org][17])

---

# 超常見 Q&A（把坑先補起來）

**Q1：為什麼我用 `gcc` 連 C++ 會各種未定義符號？**
因為它沒有幫你自動連到 C++ 標準函式庫。**用 `g++`** 來編譯/連結 C++。([gcc.gnu.org][7])

**Q2：`-I`、`-L`、`-l` 到底差在哪？**

* `-I`：告訴**預處理器**去哪找標頭。([gcc.gnu.org][11])
* `-L`、`-l`：告訴**連結器**去哪找、要連哪個庫。([gcc.gnu.org][13])

**Q3：Debug 要加什麼？Release 要加什麼？**

* Debug：`-g -O0`（GCC/Clang），MSVC 用 `/Zi` + 連結時 `/DEBUG`。([gcc.gnu.org][6])
* Release：`-O2` 或 `-O3`；MSVC 用 `/O2`。([gcc.gnu.org][6])

**Q4：Windows 的命令列去哪開？**
Visual Studio 的 **Developer Command Prompt/PowerShell**（從 VS 功能表或開始選單叫出）。([Microsoft Learn][4])

**Q5：macOS 沒有 `clang++`？**
通常安裝 **Xcode Command Line Tools** 後就有：`xcode-select --install`。([Apple Developer][18])

---

# 一頁小抄（Cheat Sheet）

* **單檔直出**

  * Linux/macOS：`g++ -std=c++14 -Wall -Wextra -O0 -g main.cpp -o app` ([gcc.gnu.org][6])
  * Windows(MSVC)：`cl /std:c++14 /W4 /Zi main.cpp` ([Microsoft Learn][8])
* **多檔案**

  * `g++ -c a.cpp -o a.o && g++ -c b.cpp -o b.o && g++ a.o b.o -o app`（或 MSVC `cl /c a.cpp && cl /c b.cpp && link a.obj b.obj`）([gcc.gnu.org][1])
* **加自家標頭與庫**

  * `-I include/`、`-L lib/`、`-lmylib`（`libmylib.a/.so`）([gcc.gnu.org][13])
* **用 pkg-config 幫你找旗標**

  * `g++ main.cpp $(pkg-config --cflags --libs sdl2)` ([people.freedesktop.org][14])
* **最小 CMake**

  * `add_executable(app src/main.cpp)` → `cmake -S . -B build && cmake --build build` ([cmake.org][17])

---

[1]: https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html "Overall Options (Using the GNU Compiler Collection (GCC))"
[2]: https://en.wikipedia.org/wiki/Object_file?utm_source=chatgpt.com "Object file"
[3]: https://clang.llvm.org/docs/UsersManual.html?utm_source=chatgpt.com "Clang Compiler User's Manual - LLVM.org"
[4]: https://learn.microsoft.com/en-us/visualstudio/ide/reference/command-prompt-powershell?view=vs-2022&utm_source=chatgpt.com "Command-line shells & prompt for developers - Visual Studio ..."
[5]: https://learn.microsoft.com/en-us/windows/wsl/install?utm_source=chatgpt.com "How to install Linux on Windows with WSL"
[6]: https://gcc.gnu.org/onlinedocs/gcc-15.2.0/gcc/index.html?utm_source=chatgpt.com "Top (Using the GNU Compiler Collection (GCC))"
[7]: https://gcc.gnu.org/onlinedocs/gcc/Invoking-GCC.html?utm_source=chatgpt.com "Invoking GCC (Using the GNU Compiler Collection (GCC))"
[8]: https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version?view=msvc-170 "/std (Specify Language Standard Version) | Microsoft Learn"
[9]: https://learn.microsoft.com/en-us/cpp/build/reference/compiler-option-warning-level?view=msvc-170&utm_source=chatgpt.com "w /W0 /W1 /W2 /W3 /W4 /Wall /Wv"
[10]: https://learn.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format?view=msvc-170&utm_source=chatgpt.com "Z7, /Zi, /ZI (Debug Information Format)"
[11]: https://gcc.gnu.org/onlinedocs/cpp/Search-Path.html "Search Path (The C Preprocessor)"
[12]: https://learn.microsoft.com/en-us/cpp/cpp/header-files-cpp?view=msvc-170&utm_source=chatgpt.com "Header files (C++)"
[13]: https://gcc.gnu.org/onlinedocs/gcc-9.2.0/gcc/Link-Options.html "Using the GNU Compiler Collection (GCC): Link Options"
[14]: https://people.freedesktop.org/~dbn/pkg-config-guide.html?utm_source=chatgpt.com "Guide to pkg-config"
[15]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/developer_guide/creating-libraries-gcc?utm_source=chatgpt.com "Chapter 17. Creating libraries with GCC | Developer Guide"
[16]: https://learn.microsoft.com/en-us/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp?view=msvc-170&utm_source=chatgpt.com "Create and Use Your Own Dynamic-Link Library (C++)"
[17]: https://cmake.org/cmake/help/latest/command/add_executable.html?utm_source=chatgpt.com "add_executable — CMake 4.1.2 Documentation"
[18]: https://developer.apple.com/forums/thread/677124?utm_source=chatgpt.com "Xcode Command Line Tools installation"
