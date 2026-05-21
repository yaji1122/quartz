---
title: "使用 Neovim 寫程式：實戰指南"
date: 2026-05-21
tags: [neovim, vim, editor, productivity, lsp, treesitter, debugging]
draft: false
---

## 摘要

Neovim 最適合寫程式的方式，不是把它當成一般文字編輯器，而是把它當成一個*模組化的編輯平台*。核心思路很簡單：

- 讓內建的編輯模型負責最重要的工作
- 在真正需要的地方補上語言智慧
- 保持設定夠小，讓你自己記得住
- 讓編輯器配合你的工作流程，而不是把你拖進一堆插件泥沼

一套實用的 Neovim 寫程式環境，通常會包含：

- **原生 LSP**：負責導覽、診斷、hover、重新命名與 code action
- **Tree-sitter**：提供更好的解析與語法感知
- **模糊搜尋器**：快速找檔案、符號與專案內容
- **格式化工具**：穩定地在儲存時自動格式化
- **Git 整合**：檢視 hunks、blame 與變更審查
- **除錯器**：當 `print` 不夠用時派上用場

目標不是 1:1 複製 IDE，而是打造一個快速、可預測、以鍵盤為中心的環境，讓改 code 的成本更低。

## 為什麼 Neovim 適合寫程式

Neovim 很適合開發，因為它介於極簡與可擴充之間。

- 它速度快，而且可腳本化。
- 你可以精準控制 keymap 與行為。
- 它支援現代語言工具，但不會遮住底層文字模型。
- 從小腳本到大型 monorepo 都能順手使用。

真正的優勢是 *edit locality*。當你熟悉 motions、text objects 與幾個 LSP 指令後，你會更少去碰滑鼠，也更少被笨重 UI 拖慢。

## 最重要的心智模型

如果你想有效率地用 Neovim 寫程式，先學這些概念：

- **Buffers**：正在編輯的文字內容
- **Windows**：同一個 buffer 的不同視窗
- **Tabs**：版面容器，不是瀏覽器分頁
- **Operators + motions**：Vim 編輯的核心語法
- **Text objects**：以 function、段落、引號、參數列表這類邏輯單位來操作
- **Registers**：可命名的剪貼簿槽位，用來 yank 與 macro
- **Macros**：可重複錄製的編輯動作

這些概念一旦通了，後面的功能就會更容易理解。

## 現代化寫程式工具鏈

一套實用的 Neovim 寫程式環境，通常長這樣：

### 1. 核心編輯器設定

先放幾個能改善日常手感的設定：

- 行號與相對行號
- 明顯的 sign column
- 合理的搜尋行為
- 符合你習慣的 split 方向
- 較短的 diagnostics 與 git signs 更新時間

### 2. 原生 LSP：語言智慧的基礎

Neovim 內建的 LSP client，是現代語言工作流的核心。

最重要的功能包括：

- **跳到定義 / 宣告 / 參考**
- **hover 文件**
- **重新命名 symbol**
- **code actions**
- **diagnostics**
- **格式化整合**
- **completion 整合**

Neovim 文件提到，LSP attach 後可以設定一些 buffer-local 預設，例如：

- `omnifunc`：插入模式 completion
- `tagfunc`：支援 `CTRL-]` 這類導覽命令
- `formatexpr`：支援 `gq` 類格式化
- `K`：顯示 hover 說明

這代表你可以把語言功能直接接到熟悉的 Vim 行為上，而不是再加一層獨立介面。

一個很好的習慣是把 LSP keymap 掛在 `LspAttach`，這樣只有在語言伺服器真的連上時，那些按鍵才會存在。

### 3. Tree-sitter：讓編輯器懂結構

Tree-sitter 讓 Neovim 對 buffer 做增量解析，並提供更精準的語法高亮與結構感知。

實務上它能幫你：

- 更準確的語法高亮
- 許多語言更好的縮排
- 結構化 text objects
- incremental selection
- 在巢狀語法中更安全地導覽程式碼

對於巢狀很深、嵌入語法很多、標點很多的語言，Tree-sitter 特別有用。

### 4. 模糊搜尋：專案導航的核心

模糊搜尋器會把找檔案與找符號從苦差事變成反射動作。

最常用的搜尋包括：

- 專案內檔案
- repo 全文搜尋
- 已開啟 buffers
- 當前檔案的 LSP symbols
- diagnostics
- 最近開啟的檔案

`Telescope` 是很常見的選擇，因為它是一個高度可擴充的 fuzzy finder，內建 pickers、sorters 與 previewers。它的 README 目前提到需要 **Neovim 0.11.7 以上**，而且要有 **LuaJIT**。

如果你還在較舊的 Neovim 上，先升級，再決定要不要採用這類插件。

### 5. 套件與工具管理

像 `mason.nvim` 這類工具管理器，可以幫你統一安裝 LSP servers、DAP adapters、linters 與 formatters。

這很重要，因為一套 coding 環境最容易失控的地方，就是每台機器的工具版本都不一樣。

理想模式是：

- 插件只裝一次
- 工具 binary 交給插件管理
- 版本要嘛固定，要嘛有計畫地更新
- 團隊設定文件保持精簡且明確

### 6. 把格式化變成自動習慣

格式化應該要無感。

像 `conform.nvim` 這種格式化插件很實用，因為它可以在儲存時格式化，必要時也能 fallback 到 LSP 格式化。這樣可以維持程式風格一致，而不會讓編輯器變成手動整理格式的地方。

建議預設：

- 儲存時自動格式化
- timeout 不要太長
- 不同 filetype 用對應 formatter
- LSP formatting 作為 fallback，而不是萬用第一選擇

### 7. Git 整合，直接在你編輯的地方看變更

`gitsigns.nvim` 可以把 Git 資訊直接帶進 buffer。

這很有價值，因為它能讓你：

- 編輯時直接看到變更行
- 快速 stage 或 reset hunks
- 不離開 buffer 就能看 blame
- 在儲存前先理解目前 diff 的上下文

對大型 codebase 來說，這是最立即有感的升級之一。

### 8. 當編輯器不夠用時：除錯

真正的開發工作遲早會需要 debugger。

`nvim-dap` 提供 Debug Adapter Protocol 支援，讓你可以：

- 啟動要除錯的應用程式
- attach 到正在跑的程序
- 設 breakpoints
- step through code
- 檢視 variables 與 stack frames

這讓 Neovim 不只是寫腳本的工具，而是完整的開發控制台。

## 寫程式的推薦流程

一個順手的日常流程會像這樣：

1. **從專案根目錄開啟專案**
2. **用 fuzzy finder 找檔案**，不要手打完整路徑
3. **用 LSP 的 definition / references 找到你要的 symbol**
4. **用 motions 與 text objects 編輯**，少做游標微調
5. **利用 diagnostics** 提早發現型別或語法問題
6. **儲存時自動格式化**，讓 style 不再是額外工作
7. **看 Git hunks** 再決定要不要 commit
8. **在 terminal split 或外部終端跑測試**
9. **當問題是行為而不是文字時，就用 DAP 除錯**

這個循環才是生產力的來源：打開、定位、修改、驗證、重複。

## 值得設定的實用 keymap

你不需要很多 keymap，只需要一小組一致的。

推薦候選：

- `gd` → 跳到 definition
- `gr` → 找 references
- `gi` → 跳到 implementation
- `K` → 顯示 hover 文件
- `<leader>rn` → rename symbol
- `<leader>ca` → code action
- `<leader>ff` → 找檔案
- `<leader>fg` → 在專案中全文搜尋
- `<leader>fb` → 切換 buffers
- `<leader>fS` → workspace symbols
- `<leader>gg` → Git status 或 blame
- `<leader>db` → 切換 breakpoint
- `<leader>dc` → 繼續除錯

重點不在按鍵本身，而在一致性。選一個 prefix，然後在每台機器上都維持同樣的心智模型。

## 一個精簡的 starter config 模式

這是一個不錯的起手式：

```lua
-- 基本編輯器手感
vim.o.number = true
vim.o.relativenumber = true
vim.o.signcolumn = "yes"
vim.o.updatetime = 200
vim.o.splitright = true
vim.o.splitbelow = true
vim.opt.ignorecase = true
vim.opt.smartcase = true

-- Buffer-local LSP mappings
vim.api.nvim_create_autocmd("LspAttach", {
  callback = function(ev)
    local map = function(mode, lhs, rhs, desc)
      vim.keymap.set(mode, lhs, rhs, { buffer = ev.buf, desc = desc })
    end

    map("n", "gd", vim.lsp.buf.definition, "跳到定義")
    map("n", "gr", vim.lsp.buf.references, "找 references")
    map("n", "gi", vim.lsp.buf.implementation, "跳到 implementation")
    map("n", "K", vim.lsp.buf.hover, "顯示說明")
    map("n", "<leader>rn", vim.lsp.buf.rename, "重新命名")
    map("n", "<leader>ca", vim.lsp.buf.code_action, "Code action")
  end,
})
```

接著再慢慢補上其他工具：

- 你常用語言的 LSP servers
- fuzzy finder
- Tree-sitter parsers
- formatter
- Git signs
- 只有在真的需要時才加的 debugger

## 應該先優化什麼

如果你剛開始用 Neovim，建議照這個順序：

### 1. 編輯速度
先學 motions、visual mode、text objects 與 macros，再看其他功能。

### 2. 語言導覽
加上 LSP，確認 definition、references、rename、diagnostics 都穩定可用。

### 3. 搜尋與專案移動
加 fuzzy finder，並且養成常用的習慣。

### 4. 格式化與 linting
讓程式風格自動化。

### 5. Git 感知
補上 inline diff 與 hunk 操作。

### 6. 除錯
等基本操作熟了，再加 DAP。

這樣可以避免一開始就把設定堆成一坨，最後反而不好用。

## 常見錯誤

### 太早裝太多插件
最快把 Neovim 變痛苦的方法，就是一次裝一堆。

先做好核心編輯、LSP、搜尋與格式化，再根據實際痛點逐步補強。

### 格式化工具互相打架
如果同一個 filetype 有多個工具都在搶著格式化，儲存時行為就會變得不可預測。

選一條主要格式化路徑，並清楚寫下 fallback。

### keymap 沒有系統
隨機 keymap 很難記。

建議按照工作類型分組：

- 導覽
- 搜尋
- 重構
- 診斷
- Git
- 除錯

### 忽略 attach 條件
語言專用 keymap 通常只應該在 LSP client 真的啟動時才存在。

`LspAttach` 就是放這些設定的好地方。

### 忽略版本需求
有些插件生態系會直接假設你用的是比較新的 Neovim。

採用前先看 README，特別是搜尋與 completion 類工具。

## 推薦的預設堆疊

如果你想先從一套實用方案開始，可以用這個組合：

- **Neovim core**：負責編輯
- **原生 LSP**：負責語言智慧
- **Tree-sitter**：負責解析與高亮
- **Telescope**：負責搜尋
- **Mason**：負責工具安裝
- **Conform**：負責格式化
- **Gitsigns**：負責 inline Git 感知
- **nvim-dap**：負責除錯

這個組合夠小，能維護；也夠強，足以應付真正的寫程式工作。

## 最後建議

最好的 Neovim 設定，是你能記得住的設定。

如果某個插件每週都沒有明顯減少你的摩擦，就刪掉它。若某個 keymap 太難記，就簡化它。若某個工作流程需要一整頁 wiki 才說得清楚，那這套設定可能已經太重了。

對大多數開發者來說，Neovim 變得好用，通常是因為它被調成了三件事：

- **快速導覽**
- **一致自動化**
- **最低心智負擔**

只要你朝這三個方向優化，編輯器就會安靜地退到背景，讓你專心寫 code。

## 來源

- [Neovim LSP docs](https://neovim.io/doc/user/lsp.html)
- [Neovim Tree-sitter docs](https://neovim.io/doc/user/treesitter.html)
- [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)
- [mason.nvim](https://github.com/williamboman/mason.nvim)
- [conform.nvim](https://github.com/stevearc/conform.nvim)
- [nvim-dap](https://github.com/mfussenegger/nvim-dap)
- [gitsigns.nvim](https://github.com/lewis6991/gitsigns.nvim)
