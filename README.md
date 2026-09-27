# Small C Compiler — C Subset to LLVM IR Compiler

> 一個用 **ANTLR4 + Java** 從零實作的 C 語言子集編譯器，將 `.c` 原始碼編譯成 **LLVM IR**，再透過 clang 產生可執行檔。內建語意分析、邊界防護，以及完整的編譯期最佳化流程(常數折疊、CSE、DCE、LICM、全域常數傳播、迴圈展開、尾呼叫優化等)，並以多組自動化回歸測試驗證正確性。

A from-scratch C-subset compiler built with ANTLR4 + Java that emits LLVM IR, complete with semantic analysis, bounds checking, and a multi-pass optimization pipeline, verified by many automated regression tests.

---

## 目錄

- [專案簡介](#專案簡介)
- [系統架構](#系統架構)
- [功能特色](#功能特色)
- [編譯期最佳化](#編譯期最佳化)
- [自訂 `##` 運算子](#自訂--運算子)
- [環境需求](#環境需求)
- [快速開始](#快速開始)
- [範例](#範例)
- [測試](#測試)
- [專案結構](#專案結構)
- [已知限制](#已知限制)

## 專案簡介

`Small C Compiler` 是一個將 C 語言子集翻譯成 LLVM IR 的編譯器，前端使用 ANTLR 4.13.2 產生 Lexer / Parser，並以 Visitor pattern 在語法樹走訪階段直接產生 IR；後端產生的 `.ll` 檔案可以直接用 `clang` 搭配隨附的 `myRuntime.c` 執行期函式庫編譯、連結並執行。

整個專案除了單純的「翻譯」之外，還實作了**語意分析**(型別檢查、隱式轉換、作用域管理)、**安全防護**(陣列邊界檢查、缺少 return 警告)以及**編譯器後端最佳化**，目標是盡量貼近真實編譯器(如 clang/GCC)的行為，而不只是滿足語法上的對應。

## 系統架構

```mermaid
flowchart LR
    A[".c 原始碼"] --> B["前處理器\n(#define / #include / 條件編譯)"]
    B --> C["ANTLR4 Lexer / Parser"]
    C --> D["AST Visitor\n語意分析 + 型別檢查"]
    D --> E["LLVM IR 產生\n(含最佳化 Pass)"]
    E --> F[".ll 檔案"]
    F --> G["clang + myRuntime.c"]
    G --> H["可執行檔"]
```

## 功能特色

| 類別 | 支援內容 |
|---|---|
| **資料型別** | int / short / long(long) / char / bool(_Bool)、unsigned 系列、float / double、`stdint.h` 固定寬度型別、struct(含位元欄位)、union、enum、typedef(含函式指標別名) |
| **陣列與指標** | 一 / 二 / 三維陣列、VLA(變長陣列)、多級指標、函式指標(含 typedef 別名、callback)、複合字面值 `(Type){...}` |
| **C99 / C11 擴充** | 具名初始化、GNU 範圍初始化 `{[0...4]=1}`、匿名 struct/union、`_Static_assert`、彈性陣列成員(FAM)、`typeof()`、`_Alignof()` |
| **運算子** | 完整四則 / 比較 / 邏輯 / 位元運算、三元運算子、複合賦值、自訂 `##` 運算子 |
| **控制流** | if/else、while、do-while、for、switch(fall-through)、goto(前 / 後向跳轉)、break/continue |
| **函式** | 多參數、遞迴 / 互遞迴、前置宣告、陣列參數退化、`int main(int argc, char *argv[])` |
| **前處理器** | `#define`(含函式型巨集、`__VA_ARGS__`)、條件編譯、`#include`、30+ 內建常數、`__LINE__/__FILE__/__func__` 等內建巨集 |
| **安全機制** | 陣列邊界靜態警告 + 動態 `abort()` 防護、缺少 return 路徑警告、`setjmp/longjmp` |
| **標準函式庫** | `printf/scanf/sprintf/fprintf`、`malloc/free/calloc/realloc`、`<math.h>`、`<string.h>`、`<assert.h>` 等常用 libc 函式,免 `#include` |

完整功能對照表與測試檔案請見 [`myCompiler_Organized_Extension_List.pdf`](./myCompiler_Organized_Extension_List.pdf) 與 [`C_Subset.pdf`](./C_Subset.pdf)。

## 編譯期最佳化

這是這個專案最花心力的部分 —— 不只是把語法翻成 IR，而是實作一條最佳化流程:

| 最佳化 | 說明 |
|---|---|
| **常數折疊 (Constant Folding)** | 純常數運算式在編譯期直接算出結果 |
| **代數化簡** | `x+0 → x`、`x*1 → x`、`x*0 → 0` 等恆等式自動消除 |
| **CSE(公共子式消除)** | 同一 Basic Block 內重複運算自動去重 |
| **DCE(死碼消除)** | 不可達區塊移除(UCE)、寫後未讀的 store 移除(DSE)、配合指標逃逸分析判斷安全性 |
| **LICM(迴圈不變式外提)** | 自動把迴圈內不變的運算搬到迴圈外執行 |
| **GCP(全域常數傳播)** | 跨 Basic Block 追蹤常數賦值,搭配 MULTI / ESCAPE 標記確保正確性 |
| **強度削減** | 乘 / 除 2 的冪次自動轉換為位移指令 |
| **窺孔最佳化 (Peephole)** | 局部 IR pattern 辨識與化簡 |
| **迴圈展開 (Loop Unrolling)** | 已知迭代次數的小型迴圈自動展開，並設有 Code Bloat 防護門檻 |
| **尾呼叫優化 (TCO)** | 自我遞迴尾呼叫改寫為 store + br 迴圈，消除額外的 call/ret 開銷 |

## 自訂 `##` 運算子

實作作業要求的自訂運算子:

```
a ## b = a^b + b^a   (a、b 皆為 float，結果為 float)
```

底層呼叫 `myRuntime.c` 中以 `powf` 實作的 `my_hashhash(float, float)`，並在 Parser 的 `multiplicativeExpression` 規則中攔截 `##` token、直接產生對應的 LLVM `call` 指令。

```c
float ha = 1.5f;
float hb = 2.5f;
float hr = ha ## hb;   // hr = 1.5^2.5 + 2.5^1.5
```

## 環境需求

- Java JDK 11+
- Clang(含 LLVM 後端)
- GNU Make
- gcc(選用，僅在用 `make gen_expected` 重新產生預期輸出時需要)

## 快速開始

```bash
# 產生 Parser → 編譯 Java → 執行所有測試
make

# 編譯並執行單一測試(若有對應 .in 檔會自動當作 stdin)
make run TEST=test1

# 只產生 LLVM IR，不連結執行
make ll TEST=test1

# 跑全部 120 組回歸測試，比對預期輸出，印出 PASS/FAIL
make check

# 清除中間產物
make clean
```

## 範例

以 [`test/test1.c`](./test/test1.c) 為例，一支做基本算術、比較與 `if-else` 的小程式:

```c
int main() {
    int a, b;
    scanf("%d", &a);
    scanf("%d", &b);

    int sum = a + b;
    printf("Sum = %d\n", sum);

    if (a > b) {
        printf("a is greater\n");
    } else if (a == b) {
        printf("a equals b\n");
    } else {
        printf("b is greater\n");
    }
    return 0;
}
```

執行 `make run TEST=test1` 後,編譯器會先輸出對應的 LLVM IR(`test1.ll`)，再交給 `clang` 連結 `myRuntime.c` 產生可執行檔並直接執行，效果與用 gcc 直接編譯這支程式完全一致。

## 測試

專案內建 **120 組測試案例**(`test/*.c`)，涵蓋從基本語法到進階擴充功能的各種情境，並以 `expected/*.txt` 紀錄用 gcc 編譯同一份原始碼所得到的「正確答案」。`make check` 會自動:

1. 用本專案的編譯器把每個 `.c` 編譯成 `.ll`
2. 用 `clang` 把 `.ll` 編譯成可執行檔並執行
3. 將實際輸出與 `expected/` 中的預期輸出逐字比對，印出 PASS / FAIL 統計

少數測試(如 `test_semantic`、`test_boundary`)是專門用來驗證**錯誤防護機制**是否正確攔截非法操作，因此「FAIL」反而代表安全機制正常運作 —— 細節請見專案內 `README`。

## 專案結構

```
.
├── myCompiler.g4          # ANTLR4 文法檔:前處理器 + Parser + 程式碼產生邏輯(核心)
├── myCompiler_test.java   # 主程式進入點
├── myRuntime.c            # 執行期函式庫(## 運算子底層實作)
├── Makefile                # 建置 / 測試 / 回歸比對
├── test/                   # 120 組測試 .c 檔與對應 stdin
├── expected/                # 各測試的預期輸出
├── README.pdf             # 編譯執行以及編譯器功能詳細說明
├── C_Subset.pdf             # 支援的 C 語言子集規格說明
└── myCompiler_Organized_Extension_List.pdf  # 擴充功能與對應測試檔案總覽
```

## 已知限制

- 不支援 `(*p)++`、`arr[i] += 1` 等複雜左值的自增 / 複合賦值
- `printf`/`scanf` 的格式字串必須是原始碼中的字面值，不能是變數
- 不支援 `long double`、`__int128`、C11 atomics
- `switch` 條件僅支援整數字面值，不支援 `enum` 常數名稱或 `float`/`long`
- 全域陣列不支援初始化清單

完整清單請見專案內 `README`(第七節「不支援的功能」)。

---

**作者**: 余沛穎 ｜ 本專案為課程作業延伸實作
