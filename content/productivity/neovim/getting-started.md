---
title: "安裝與初始設定"
date: 2026-05-21
tags: [neovim, vim, editor, productivity]
draft: false
---

## 先把 Neovim 裝起來

如果你只是想先開始用，不需要一口氣把整套插件架好。先確保 Neovim 能正常啟動、能讀到設定檔，就已經成功一半了。

### 安裝方式

常見安裝方式有幾種：

- **macOS**：`brew install neovim`
- **Ubuntu / Debian**：使用套件管理器安裝，或直接抓官方 release
- **Arch Linux**：`pacman -S neovim`
- **其他平台**：也可以直接下載 Neovim 官方發佈版

安裝完先確認版本：

```bash
nvim --version
```

如果能看到版本資訊，代表基本安裝沒問題。

## 第一個設定檔

Neovim 的主要設定檔通常是：

```text
~/.config/nvim/init.lua
```

你可以先建立這個檔案，從最小可用設定開始，不必一次寫很多東西。

### 最小設定範例

```lua
vim.g.mapleader = " "

vim.opt.number = true
vim.opt.relativenumber = true
vim.opt.mouse = "a"
vim.opt.clipboard = "unnamedplus"
vim.opt.termguicolors = true
vim.opt.expandtab = true
vim.opt.shiftwidth = 2
vim.opt.tabstop = 2
vim.opt.smartindent = true
vim.opt.wrap = false
vim.opt.undofile = true
```

這些設定的目的很單純：

- **number / relativenumber**：更容易定位行號
- **mouse = "a"**：需要時可以先用滑鼠輔助
- **clipboard**：方便和系統剪貼簿互通
- **termguicolors**：讓顏色主題顯示正常
- **indent 設定**：維持一致縮排
- **undofile**：關掉編輯器後還能保留復原歷史

## 建議的初始順序

如果你是第一次接觸 Neovim，可以照這個順序來：

1. 先安裝 Neovim
2. 建立 `init.lua`
3. 先放幾個基本 `vim.opt` 設定
4. 熟悉基本移動、寫入、關閉、分割視窗
5. 再慢慢加插件

這樣比較不會一開始就被插件和設定淹沒。

## 初期不要急著做的事

剛開始時，通常不需要馬上：

- 建一大堆 plugin
- 做超複雜的自動化設定
- 把所有 keymap 一次改完
- 追求完全客製化的 UI

先讓編輯器穩定、順手、可預期，後面再逐步擴充，會比較容易維持。

## 下一步

如果你已經完成安裝與最小設定，可以接著看：

- [概覽：Neovim 為什麼適合寫程式](overview/)
- [LSP 與 Tree-sitter](lsp-tree-sitter/)
- [搜尋、格式化與 Git 整合](tooling/)
- [工作流、keymap 與設定策略](workflow-config/)
- [除錯與推薦預設堆疊](debugging-stack/)
