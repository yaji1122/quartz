---
title: "搜尋、格式化與 Git 整合"
date: 2026-05-21
tags: [neovim, telescope, mason, conform, gitsigns, productivity]
draft: false
---

## 模糊搜尋：專案導航的核心

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

## 套件與工具管理

像 `mason.nvim` 這類工具管理器，可以幫你統一安裝 LSP servers、DAP adapters、linters 與 formatters。

這很重要，因為一套 coding 環境最容易失控的地方，就是每台機器的工具版本都不一樣。

理想模式是：

- 插件只裝一次
- 工具 binary 交給插件管理
- 版本要嘛固定，要嘛有計畫地更新
- 團隊設定文件保持精簡且明確

## 格式化：把整潔變成自動化

格式化應該要無感。

像 `conform.nvim` 這種格式化插件很實用，因為它可以在儲存時格式化，必要時也能 fallback 到 LSP 格式化。這樣可以維持程式風格一致，而不會讓編輯器變成手動整理格式的地方。

建議預設：

- 儲存時自動格式化
- timeout 不要太長
- 不同 filetype 用對應 formatter
- LSP formatting 作為 fallback，而不是萬用第一選擇

## Git 整合：直接看變更

`gitsigns.nvim` 可以把 Git 資訊直接帶進 buffer。

這很有價值，因為它能讓你：

- 編輯時直接看到變更行
- 快速 stage 或 reset hunks
- 不離開 buffer 就能看 blame
- 在儲存前先理解目前 diff 的上下文

對大型 codebase 來說，這是最立即有感的升級之一。

## 下一步

如果你想把日常操作與設定策略整理好，可以看：

- [工作流、keymap 與設定策略](../workflow-config/)
- [除錯與推薦預設堆疊](../debugging-stack/)
