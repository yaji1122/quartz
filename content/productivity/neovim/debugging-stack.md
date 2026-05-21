---
title: "除錯與推薦預設堆疊"
date: 2026-05-21
tags: [neovim, dap, debugging, productivity]
draft: false
---

## 當編輯器不夠用時：除錯

真正的開發工作遲早會需要 debugger。

`nvim-dap` 提供 Debug Adapter Protocol 支援，讓你可以：

- 啟動要除錯的應用程式
- attach 到正在跑的程序
- 設 breakpoints
- step through code
- 檢視 variables 與 stack frames

這讓 Neovim 不只是寫腳本的工具，而是完整的開發控制台。

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
