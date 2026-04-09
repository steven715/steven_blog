# C++ 專案框架規範

本文件定義 C++ 服務型專案的目錄結構、設計慣例、文件框架與構建規則。
所有新建專案與既有專案的重構均應遵照本規範。

---

## 1. 專案目錄結構

```
{PROJECT_NAME}/
├── src/                          # 所有原始碼（.cpp + .h 同目錄）
│   ├── app/                      # 應用程式組裝層
│   │   └── application.cpp/.h    #   元件初始化、生命週期管理、信號處理
│   ├── main.cpp                  #   程式入口，僅呼叫 Application::run()
│   ├── config/                   # 配置載入與驗證
│   ├── manager/                  # 業務物件管理器（擁有資料生命週期）
│   ├── model/                    # 貧血資料結構（struct）
│   ├── relay/                    # 網路通訊、協議適配
│   ├── utils/                    # 通用工具（日誌、時間、字串處理）
│   └── {domain_module}/          # 依業務領域擴展的模組
├── tests/
│   ├── unit/                     # 單元測試（鏡射 src/ 結構）
│   │   ├── config/
│   │   ├── manager/
│   │   ├── model/
│   │   └── {domain_module}/
│   ├── integration/              # 整合測試（涉及外部依賴）
│   └── CMakeLists.txt
├── docs/
│   ├── design/                   # Design Doc（含 Context、Goals/Non-goals）
│   ├── adr/                      # Architecture Decision Records
│   │   └── NNNN-{決策標題}.md
│   └── diagrams/                 # 系統互動關係圖（元件圖、Container 圖）
├── scripts/                      # CI/CD、效能分析、sanitizer 腳本
├── cmake/                        # CMake 輔助模組（FindXxx.cmake 等）
├── third_party/                  # 第三方依賴（若非由套件管理器處理）
├── .gitignore
├── CMakeLists.txt                # 頂層構建配置
├── {project_name}.cfg            # 運行時配置檔
└── README.md
```

### 1.1 目錄職責說明

- `src/` 下按**功能模組**分子目錄，每個子目錄內 `.h` 與 `.cpp` 並置，不做 `include/` 與 `src/` 分離。分離 header 的做法僅適用於對外發布的 library；服務型應用程式的所有 header 皆為內部使用。
- `app/` 是唯一知道所有模組存在的地方。它負責組裝元件、注入依賴、啟動事件迴圈。其他模組彼此不直接建構對方——它們透過介面（callback、抽象基底類別）協作。
- `main.cpp` 不包含業務邏輯，僅作為程式入口點。典型內容不超過 5 行。
- `tests/unit/` 的子目錄結構鏡射 `src/`，測試檔命名為 `{被測檔名}_test.cpp`。
- `tests/integration/` 放置需要外部依賴（Kafka、MongoDB、網路）的測試。
- `docs/adr/` 中的檔案以流水號加標題命名，如 `0001-eventbus-集中式而非點對點.md`。

### 1.2 不允許出現的結構

- `src/` 下不放測試檔案。
- `main.cpp` 不出現業務邏輯代碼（初始化配置、建構業務物件、啟動迴圈皆屬 `app/` 職責）。
- 不建立頂層 `include/` 目錄（除非專案同時產出對外 library）。
- `build/` 目錄不進版本控制。

---

## 2. 構建架構：Thin-Main + Library 模式

所有專案採用三 target 構建架構，以確保業務邏輯可獨立測試。

### 2.1 CMake Target 定義

```cmake
# ── 1. 業務邏輯 library（所有原始碼，除 main.cpp 外）──
add_library(${PROJECT_NAME}_core
    src/app/application.cpp
    src/config/config_loader.cpp
    src/manager/quote_manager.cpp
    src/model/quote_data.cpp
    # ... 所有業務邏輯原始碼
)
target_include_directories(${PROJECT_NAME}_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src)

# ── 2. 可執行檔（僅 main.cpp，連結 library）──
add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_core)

# ── 3. 測試可執行檔（連結同一個 library）──
enable_testing()
add_subdirectory(tests)
```

### 2.2 main.cpp 標準寫法

```cpp
#include "app/application.h"

int main(int argc, char* argv[]) {
    Application app(argc, argv);
    return app.run();
}
```

### 2.3 測試的 CMakeLists.txt

```cmake
find_package(GTest REQUIRED)

# 單元測試
add_executable(${PROJECT_NAME}_unit_tests
    unit/manager/quote_manager_test.cpp
    unit/model/quote_data_test.cpp
)
target_link_libraries(${PROJECT_NAME}_unit_tests
    PRIVATE ${PROJECT_NAME}_core GTest::gtest_main
)
gtest_discover_tests(${PROJECT_NAME}_unit_tests TEST_PREFIX "unit.")

# 整合測試
add_executable(${PROJECT_NAME}_integration_tests
    integration/relay/relay_connection_test.cpp
)
target_link_libraries(${PROJECT_NAME}_integration_tests
    PRIVATE ${PROJECT_NAME}_core GTest::gtest_main
)
gtest_discover_tests(${PROJECT_NAME}_integration_tests TEST_PREFIX "integration.")
```

### 2.4 測試執行策略

| 階段 | 執行範圍 | 命令 | 備註 |
|------|----------|------|------|
| Pre-commit | 僅單元測試 | `ctest --test-dir build -L unit` | 必須全部通過才能 commit |
| CI Build | 全部測試 | `ctest --test-dir build` | 包含整合測試 |
| CI Analysis | Sanitizer + Perf | 見 scripts/ | 通過標準後才能發布 |

---

## 3. 型別設計慣例

### 3.1 struct vs class 區分

| 特徵 | struct | class |
|------|--------|-------|
| 用途 | 貧血資料模型（Data/Info/Snapshot） | 有行為的業務物件 |
| 成員 | 全 public 資料欄位 | private 資料 + public 介面 |
| 生命週期 | 由外部（manager/owner）管理 | 自身管理或由明確的 owner 管理 |
| 範例 | `QuoteData`, `OrderInfo`, `RiskSnapshot` | `QuoteManager`, `RiskEvaluator`, `StateMachine` |

```cpp
// 貧血資料結構 —— 用 struct
struct QuoteData {
    std::string symbol;
    double bid_price;
    double ask_price;
    int64_t timestamp;
};

// 有行為的業務物件 —— 用 class
class QuoteManager {
public:
    void update(const QuoteData& quote);
    const QuoteData* get(const std::string& symbol) const;
private:
    std::unordered_map<std::string, QuoteData> quotes_;
};
```

### 3.2 命名慣例

- struct 名稱使用資料語意後綴：`Data`, `Info`, `Snapshot`, `Config`, `Entry`。
- class 名稱使用行為語意：`Manager`, `Evaluator`, `Processor`, `Publisher`, `StateMachine`。
- 測試檔名：`{原始檔名}_test.cpp`（如 `quote_manager_test.cpp`）。
- ADR 檔名：`NNNN-{簡短標題}.md`（如 `0001-eventbus-採用集中式設計.md`）。

### 3.3 模組間通訊原則

- 模組間互動僅透過 **callback** 或 **事件匯流排（EventBus）**，不允許模組直接建構或持有另一個模組的實例。
- 資料擁有者提供**唯讀存取介面**（回傳 `const&` 或 `const*`），業務模組透過該介面取得資料後獨立運算。
- 寫入操作限制為**單一維度**——一次寫入只修改一個業務物件的狀態，不跨物件做聯合寫入。
- 中介軟體（EventBus、網路連線等）的生命週期由 `Application` 統一管理，業務模組不負責中介軟體的建構與銷毀。

---

## 4. 文件框架

每個專案至少包含以下四層文件，各層覆蓋不同的認知需求。

### 4.1 Design Doc（`docs/design/`）

採用精簡版 Google Design Doc 結構，一個專案的每個重要子系統或設計決策對應一份文件。

```markdown
# {子系統名稱} Design Doc

## Context and Scope
- 本系統在更大技術版圖中的位置
- 上下游系統與資料流方向
- 附一張 Container 層級圖（系統間關係）

## Goals
- 明確的功能目標與可量化的效能指標

## Non-goals
- 合理地可能被期望為本系統職責、但被刻意排除的事項
- 每條 Non-goal 說明由誰負責

## Design
- 敘事：每個角色的職責（who does what）
- 圖：元件互動關係（who talks to whom）
- 關鍵取捨的推理過程

## Alternatives Considered
- 被考慮但未採用的方案及其理由
```

### 4.2 Architecture Decision Records（`docs/adr/`）

記錄單一設計決策的 why。當系統的現有設計被質疑或需要變更時，ADR 提供回溯的依據。

```markdown
# ADR-NNNN: {決策標題}

## 狀態
Accepted / Superseded by ADR-XXXX / Deprecated

## 背景
做這個決策時的系統狀態與面臨的問題。

## 決策
我們決定 {做法}。

## 理由
為什麼選擇這個方案而非其他方案。

## 後果
這個決策帶來的正面與負面影響。

## 重新審視條件
當以下條件發生時，應重新評估本決策：
- {條件 1}
- {條件 2}
```

### 4.3 系統互動圖（`docs/diagrams/`）

圖的職責是表達**互動關係**（who talks to whom），不表達職責（who does what，由敘事負責）。

規則：
- 每張圖只服務**一個縮放層級**。Container 圖顯示系統間關係，Component 圖顯示一個系統內部的模組關係，不混合。
- 箭頭用顏色區分語意（如：紅色=推送，綠色=訂閱）。
- 圖中每個元件只列出職責動詞（推送、訂閱、新增、刪除），不放實作細節。

### 4.4 README.md

專案根目錄的 README 是最外層的入口文件，內容包含：
- 一句話描述本專案是什麼、做什麼。
- 構建與運行指令。
- 指向 `docs/design/`、`docs/adr/`、`docs/diagrams/` 的連結索引。

---

## 5. 設計原則

以下原則適用於所有專案的架構與實作決策。

### 5.1 邊界設計

- **資料歸資料，行為歸行為。** struct 不包含業務邏輯；class 不暴露內部資料結構。
- **業務物件擁有特定職責。** 一個 class 只負責一個明確的業務域（如倉位管理、規則判斷、盈虧計算），不做跨域操作。
- **寫入限制為單一維度。** 避免一個操作同時修改多個業務物件的狀態，以限縮錯誤影響範圍。

### 5.2 錯誤處理

- **偏好明顯的失敗而非似是而非的錯誤。** 系統在異常情況下應快速且明確地失敗（crash / 回傳錯誤碼），而非默默地進入一個看起來正常但資料不一致的狀態。
- 狀態機中對競爭寫入事件採用 **first-in-wins** 搭配終端狀態，不做合併。

### 5.3 產品化標準

一個系統被視為「產品化完成」的判斷標準：**一個有能力但不熟悉本系統的工程師，能在合理時間內接手運維與演進。** 這要求：
- 知識存在於系統與文件中，而非人的腦中。
- Design Doc + ADR + 互動圖 + 測試共同構成接手所需的完整資訊。

### 5.4 效能關鍵路徑

在效能敏感的路徑上（如行情處理、撮合引擎）：
- 偏好 lock-free 資料結構與 zero-allocation 策略。
- 使用 `std::atomic`、SeqLock 等低開銷同步原語。
- 避免在熱路徑上做序列化（如 BSON/JSON encode）或同步 I/O。
- 效能分析工具：`perf` + flame graph、ThreadSanitizer、AddressSanitizer。

---

## 6. CI/CD 規劃（漸進實施）

### 階段一：測試基礎設施（當前目標）
- 建立 `tests/unit/` 結構，引入 Google Test。
- CMake 配置三 target 構建（core library + executable + tests）。
- 至少對核心業務模組撰寫單元測試。

### 階段二：Pre-commit 守門
- 配置 pre-commit hook，執行 `ctest -L unit`。
- 單元測試未全部通過則阻止 commit。
- 引入 `clang-format` 確保代碼風格一致。

### 階段三：CI Pipeline
- 自動化構建 + 全量測試（unit + integration）。
- 執行 AddressSanitizer / ThreadSanitizer 檢查。
- 執行 `perf` 效能基準測試，與歷史數據比較。
- 通過所有標準後才標記為可發布。

---

## 7. 快速參考：新建專案 Checklist

啟動一個新專案時，按以下順序建立：

1. 建立目錄結構（`src/`, `tests/`, `docs/`, `scripts/`）。
2. 撰寫頂層 `CMakeLists.txt`，定義三個 target。
3. 建立 `src/main.cpp`（thin entry point）和 `src/app/application.h/.cpp`。
4. 撰寫 `docs/design/{subsystem}_design.md`，至少包含 Context、Goals/Non-goals。
5. 為第一個核心模組撰寫 `_test.cpp`，確認 GTest 運行正常。
6. 建立 `README.md`，包含構建指令與文件索引。
7. 配置 `.gitignore`（排除 `build/`、IDE 配置檔）。
