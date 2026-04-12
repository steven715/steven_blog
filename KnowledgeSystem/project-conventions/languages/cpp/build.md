# C++ Build Configuration

CMake 構建架構與 Thin-Main 模式。

<!--
📌 預期擴展：
- 第三方依賴管理策略（static vs dynamic linking 的選用）
- Symbol visibility 控制（pimpl、__attribute__((visibility)))
- Cross-compilation 配置（CentOS 7 → Ubuntu 24.04 遷移經驗）
- CMake preset 的使用
- Package 產出規範（RPM / DEB / Docker image）
-->

---

## 三 Target 構建架構

所有專案採用 core library + executable + tests 的三 target 結構，
確保業務邏輯可獨立測試。

```cmake
# 1. 業務邏輯 library（所有原始碼，除 main.cpp 外）
add_library(${PROJECT_NAME}_core
    src/app/application.cpp
    # ... 所有業務邏輯原始碼
)
target_include_directories(${PROJECT_NAME}_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/src)

# 2. 可執行檔（僅 main.cpp）
add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_core)

# 3. 測試（連結同一個 library）
enable_testing()
add_subdirectory(tests)
```

## Thin-Main 標準寫法

```cpp
#include "app/application.h"

int main(int argc, char* argv[]) {
    Application app(argc, argv);
    return app.run();
}
```

main.cpp 不超過 5 行，不包含任何業務邏輯。
初始化配置、建構業務物件、啟動迴圈皆屬 `app/` 職責。
