---
title: "LSP 與 Tree-sitter"
date: 2026-05-21
tags: [neovim, lsp, treesitter, productivity]
draft: false
---

## 原生 LSP：語言智慧的基礎

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

## Tree-sitter：讓編輯器懂結構

Tree-sitter 讓 Neovim 對 buffer 做增量解析，並提供更精準的語法高亮與結構感知。

實務上它能幫你：

- 更準確的語法高亮
- 許多語言更好的縮排
- 結構化 text objects
- incremental selection
- 在巢狀語法中更安全地導覽程式碼

對於巢狀很深、嵌入語法很多、標點很多的語言，Tree-sitter 特別有用。

## 什麼時候最有感

這兩個工具搭配時，通常在這些地方最有感：

- 需要快速看懂大型檔案
- 需要頻繁跳 symbol
- 需要重構命名
- 需要在複雜語法中做精準選取

## 下一步

如果你想把工作流補完整，可以接著看：

- [搜尋、格式化與 Git 整合](../tooling/)
- [工作流、keymap 與設定策略](../workflow-config/)
