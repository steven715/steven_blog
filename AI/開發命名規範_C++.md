# 開發命名規範 - C++

### 目的

- 優化可讀性
- 統一命名規則

## Namespace

- Base on the project name
- Do not use `using-directives` (e.g. `using namespace foo` )

### Namespace Names

- Namespace names are all lower-case, with words separated by underscores

## Local Variables

- Place a function’s variables in the narrowest scope possible, and initialize variables in the declaration.
- e.g. `int i`, `std::vector<int> v` , `Foo f`

### Variable Names

- Using `snake_case` (all lowercase, with underscores between words)
- Common Variable Names, e.g.`std::string table_name`
- Class Data Member with a trailing underscore, e.g. `std::string people_name_`

## Nonmember, Static Member, and Global Functions

- Placing nonmember functions in a namespace

## General Naming Rules

Optimize for readability using names that would be clear even to people on a different team

- Use names that describe the purpose or intent of the object.
- Do not worry about horizontal space as it is far more important to make your code immediately understandable by a new reader.
- For names written in mixed case, in which the first letter of each word is capitalized, prefer to capitalize abbreviations as single words, e.g., `StartRpc()` rather than `StartRPC()`.

# 參考資料

- https://google.github.io/styleguide/cppguide.html