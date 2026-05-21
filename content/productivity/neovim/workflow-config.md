---
title: "工作流、keymap 與設定策略"
date: 2026-05-21
tags: [neovim, workflow, keymap, config, productivity]
draft: false
---

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
