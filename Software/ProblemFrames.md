# Problem Frames

問題框架是一種有關軟體上的需求分析 (Problem frames is an approach to software requirements analysis.)

## Fundanmental philosophy

底層邏輯是為了電腦軟體(computer software)收集需求(gathering requirements)並決定實作框架(creating specifications)

- 最好的需求分析是透過平行過程(through a process of parallel)，解藕用戶的需求(decomposition of user requirements)
- 用戶的需求只關心軟體與真實世界的關係(relationships in the read world)，翻譯成專業術語就是 **application domain** ，他們不關心軟體系統是如何組成或是有什麼接口等

## Three sets of conceptual tools

### Tools for describing specific problems

Concepts for describing specific problems include

- phenomena: 用戶關心、可描述的事實
- problem context 
- problem domain 
- solution domain (machine)
- shared phenomena (which exist in domain interface)
- domian reuquirements (which exist in the problem domain)
- specifications (which exist at the problem domain:machine interface)

### Tools for describing classes of problems (problem frames)

問題框架中也有相似的類別，類似於軟體中的設計模式，一種屬於同性質的問題，被大家歸納出一套有體系的方法或原則來處理

在一個問題框架中，領域(domain)可能會有一個泛型的名字，並由幾個關鍵特性的形容詞組成，其中有可能是
- casual: 因果關係，擁有可預測性的事件
- biddable: 能要求，但結果不如預期的關係

### A list of recognized classes of problems (problem frames)

1. required behavior
2. commanded behavior
3. information display
4. simple workpiece
5. transformation

## Describing problems

### The problem context

問題分析(problem analysis)中認為軟體應用(software application)是一種軟體機器(software machine)，軟體開發專案目標是改變問題邊界(problem context)透過建立軟體機器到問題邊界裡，以達到預期的效果。

- Application domain: 再問題邊界(problem context)下，除了軟體機器(machine)外的其他部分，這部分是形成(form)問題邊界(problem context)。

### The context diagram

A domain is simply a part of the real world that we are interested in. It consists of **phenomena**

- individuals: 表演票、場次、表演
- event: 買票、看有哪些空座位
- states of affeirs: 座位被訂走、空閒
- relationships: 票跟座位再加上那一天是綁定的 (同一時段只能有一個座位跟一張票)
- behaviors: 開演10分鐘後不再售票

A domain interface is an area where domains connect and communicate. An interface is a place where domains partially overlap, so that phenomena in the interface are **shared phenomena** - they exist in both of the overlapping domains.

- 觀眾買了A3的票:
  - 觀眾: 點了螢幕上的座位
  - 售票系統(domain): 識別座位是A3

### Problem diagram





## 個人理解

- 一個現實的問題，我想要賣一疊表演票
- 問題邊界(context): 真實世界已經有的部分，以及這些部分有什麼樣的現象，觀眾選擇場次、表演時間、表演等
- 形成(form): 這個問題邊界所擁有的規則、條件，表演票的使用時間、表演票如何被移轉、座位如何被使用
  
## wiki

https://en.wikipedia.org/wiki/Problem_frames_approach
