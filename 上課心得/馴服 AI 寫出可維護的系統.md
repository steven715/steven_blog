# Day 1 1/17
## 軟體 single source of truth

- 認知的唯一真相
  - Source code
  - Document

- 軟體開發本質還是 **specfication driven development** => Document
  - 因為客戶最終反應的是文件上說明的內容

## AI Coding 挑戰 

如何讓不確定的生成過程，收斂為可驗證、可重現的結果?

**收斂** (determinism) 是關鍵 => "How to document knowledge that AI can use it?"

### 最不信任AI的地方

1. Thread safe code
2. Performance code
3. Bussiness logic
4. Sercurity issues

#### 可解決部分

1. Bussiness logic
   1. 人寫完整的測試案例 (before LLM)
2. Sercurity
   1. 寫好規範的prompt，寫出AI不可觸碰的行為 (before LLM and after LLM)
3. Performance
   1. 先定義好SLA，讓AI執行相關的測試案例，來看是否有符合 (after LLM)

#### 不可解決部分

1. Thread safe
   1. 如何讓AI寫出thread safe，透過語意一致性，人需要介入處理

## 如何限制AI的non-determinism

- 怎麼讓AI證明他是對的 ?
- 怎麼知道他有完全照著prompt做 ?
- 透過MCP，讓AI知道現況

## 以前的MDA，只寫spec會失敗的原因

表達能力來說，source code比DML還強，自然語言也是，會有歧異性

## AI Coding 中如何消除這些歧異性

透過prompt以及不斷迭代每次的prompt，每次AI犯錯，讓AI紀錄該錯誤，並時不時提醒AI回顧那些犯錯紀錄

## 設計概念

設計就是決定要設計什麼

- Form: 外觀，solution, product
- Context: 不是，solution所涵蓋的
- Force: 問題限制條件

## Context類型

- 先決定context，form就在其中
- 如果可以窮舉context，form就不再是設計層面的問題，而主要成為實作上的技術問題

## 軟體開發的設計

- Code first (code is form)
- test first (TDD, test is context)

## AI Coding 要告訴 AI 哪些 context

- requirement
- 驗收條件
- coding style
- glossary
- code architecture

".dev/specs/vo/vo-spec.json 是 Value Object 規格，幫我依此生成 XXX.java 符合此風格，幫我寫一個vo-sub-agent.md prompt 達成這個任務" 

## AI Coding Precondition

- 團隊成熟度(規範、代碼風格)
- AI Coding的品質取決於寫code的人

**軟體架構跟功能無關，主要解決non-functional的問題**

- **Problem 跟 Solution 在表達能力要匹配**
  - AI 拿 regular expression 去解決 language 級別的問題
