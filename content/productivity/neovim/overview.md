---
title: "概覽：Neovim 為什麼適合寫程式"
date: 2026-05-21
tags: [neovim, vim, editor, productivity]
draft: false
---

## 摘要

Neovim 最適合寫程式的方式，不是把它當成一般文字編輯器，而是把它當成一個 *模組化的編輯平台*。

核心思路很簡單：

- 讓內建的編輯模型負責最重要的工作
- 在真正需要的地方補上語言智慧
- 保持設定夠小，讓你自己記得住
- 讓編輯器配合你的工作流程，而不是把你拖進一堆插件泥沼

## 為什麼 Neovim 適合開發

Neovim 很適合開發，因為它介於極簡與可擴充之間。

- 它速度快，而且可腳本化
- 你可以精準控制 keymap 與行為
- 它支援現代語言工具，但不會遮住底層文字模型
- 從小腳本到大型 monorepo 都能順手使用

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

## 下一步

如果你已經有基本概念，可以接著看：

- [LSP 與 Tree-sitter](../lsp-tree-sitter/)
- [搜尋、格式化與 Git 整合](../tooling/)
- [工作流、keymap 與設定策略](../workflow-config/)
- [除錯與推薦預設堆疊](../debugging-stack/)
