# Day 2 1/18

- 給AI的prompt (json or yaml) 可以讓AI從source code去整理出來

## 如何將 Real World Problem 收斂

1. 補領域知識
   1. 更知道要做什麼
2. 找問題分析的方法 (like Problem frames)
   1. 更好的定義出邊界

## 如何增加測試的可靠性

- Mutation Testing: 變異測試，透過將業務程式代碼寫錯，再跑測試案例，理論上測試案例要都失敗才合理。

## Pattern Language

- 如何設計自己的 Pattern Language

## 需求 vs 規格

- 需求 Requirement: 世界應該變成什麼樣子
- 規格 Specification: 機器承諾的行為，用以實現該需求

## 如何在 MultiUser的系統去規範 Race Condition

- DDD的aggregate本身就有做同步性的議題

## DDD 系統分三類

- Generic: 通用類型，像是RBAC
- Supporting: 幫助核心的部分，報表系統
- Core: 業務核心，考試系統本身

## 如何識別一個人的水平

1. 聽他的問題，是否表達清晰，是否邏輯一致性
2. 他的回答，我是否能馬上想出反例，如果可以，代表他沒有真的想過或想完問題
