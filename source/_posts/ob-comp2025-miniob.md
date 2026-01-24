---
mathjax: true
title: OceanBase 2025 初赛 MiniOB 开发记录
date: 2026-01-24 16:01:52
tags:
  - MiniOB
  - Oceanbase数据库大赛
category: 'Oceanbase数据库大赛'
typora-root-url: ob-comp2025-miniob
---
初赛于2025年10月20日开始，10月24日下午截止报名，很遗憾没能在报名截止之前做到满分（笑）（排第一的那支队伍做到了）。

![初赛成绩](miniob-score.png)

代码在2025年7~8月就已经开始写了，当时做的是2024年的训练营。

## 链接

初赛代码仓库：[BeingTrappedInDB/miniob-competition2025](https://github.com/BeingTrappedInDB/miniob-competition2025)

2025年训练营：<https://open.oceanbase.com/train/detail/17?questionId=600060>

比赛主页和榜单：<https://open.oceanbase.com/competition/2025>

## 从 2024 到 2025：MiniOB 的演进与挑战

相比 2024 年，2025 年的赛题在原有基础上新增了若干题目，同时也对部分旧题补充了新的测试用例。这导致不少在 2024 年训练营中可以通过的实现，在 2025 年环境下已无法通过。
从整体来看，MiniOB 的代码基础相当扎实，各个模块划分清晰，但代码体量并不算大，这反而为参赛者在架构设计与性能取舍上提供了较大的自由空间。

初赛阶段的核心目标依然是**功能正确性**，对性能的要求并不算苛刻。
在 2024 年的 ann-benchmark 赛题中，虽然引入了性能约束，但实际上只要找准一两个关键瓶颈并加以优化，再配合 `release` 编译，基本就能满足要求。
而 2025 年的赛题则明显将重心转向了**内存开销限制**，这大概率是为了强制选手为 `big-order-by` 等场景实现**外部排序**机制。相较之下，对时间性能的要求反而被弱化了。

从更大的背景来看，MiniOB 目前实际上承载了多类教学与竞赛场景的实验代码，包括：

* OceanBase 数据库大赛：行存 + 火山模型 + 功能实现
* CCF 数据库暑期学校：PAX 存储 + 向量化执行模型 + OLAP 性能优化
* 部分高校课程作业：LSM 树存储引擎

但近几年 OceanBase 数据库大赛仅使用了**行存格式 + 火山执行模型**这一路径，其它模块（如向量化执行引擎、LSM 树相关代码）在比赛中完全不会被用到。
因此，从工程简化的角度来看，参赛者甚至可以在比赛开始前直接移除这些无关模块，以减少理解成本，也有助于聚焦真正与赛题相关的核心逻辑。

## 设计思路

### 类型系统

#### `Value`类

MiniOB 中用于存放值的 `Value` 类型基于 `union` 实现，并采用了**手动内存管理**的方式。这种设计在早期 C++ 中较为常见，但随着代码规模的增长，其复杂性和维护成本也逐渐显现出来。

从功能实现上看，当前 `Value` 的逻辑还存在一定程度的**冗余与重复**。例如：

* 类型转换主要集中在 `cast_to` 中实现；
* 但在 `get_类型` 系列函数中，又再次内嵌了一套转换逻辑；
* 实际上，在调用这些 `get_类型` 函数之前，绝大多数场景都会先进行类型检查，因此这里的二次转换几乎永远不会生效，反而成为**冗余分支**，带来额外的分支预测开销；
* 更不一致的是，在 `Value::get_string_t()` 中又完全不做类型转换，进一步加剧了接口语义的不统一。

这类“防御式但不可达”的代码，一方面增加了阅读和维护难度，另一方面也在性能路径上埋下了不必要的分支成本。

考虑到后续比赛环境中使用的 C++ 标准基本都已不低于 **C++20**，完全可以采用更加现代、安全、语义清晰的方式来重构 `Value`，例如使用 `std::variant` 来替代 `union`，由类型系统接管对象生命周期和访问约束，从根源上消除手动管理与重复分支的问题：

```cpp
class Value final
{
public:
  using value_t = std::variant<std::nullptr_t, string, vector<float>, int, float, Date, bool, UndefinedType>;
private:
  value_t value_;
};
```

相比原先`Value`的结构：

```cpp
class Value final
{
private:
  AttrType attr_type_ = AttrType::UNDEFINED;
  int      length_    = 0;
  union Val
  {
    int32_t int_value_;
    float   float_value_;
    bool    bool_value_;
    char   *pointer_value_;
  } value_ = {.int_value_ = 0};
  bool own_data_ = false;
};
```

在使用了`std::variant`之后，`Value`的不少成员函数都能做些简化：

1. 复制、移动构造函数都不需要专门去实现了，避免手动管理内存或所有权：

```cpp
Value(const Value &other) = default;
Value(Value &&other)      = default;
Value &operator=(const Value &other) = default;
Value &operator=(Value &&other)      = default;
```

1. `get_类型`和`set_类型`也都能得到简化：

```cpp
int                  get_int() const { return std::get<int>(value_); }
float                get_float() const { return std::get<float>(value_); }
Date                 get_date() const { return std::get<Date>(value_); }
const string        &get_string() const { return std::get<string>(value_); }
string              &get_string() { return std::get<string>(value_); }
const vector<float> &get_vector() const { return std::get<vector<float>>(value_); }
vector<float>       &get_vector() { return std::get<vector<float>>(value_); }
bool                 get_boolean() const { return std::get<bool>(value_); }

void set_null() { value_ = nullptr; }
void set_int(int val) { value_ = val; }
void set_float(float val) { value_ = val; }
void set_date(Date val) { value_ = val; }
void set_string(string_view val) { value_ = string(val); }
void set_string(string &&val) { value_ = std::move(val); }
void set_vector(span<const float> vec) { value_ = vector<float>(vec.begin(), vec.end()); }
void set_vector(vector<float> &&vec) { value_ = std::move(vec); }
void set_boolean(bool val) { value_ = val; }
void set_undefined() { value_ = UndefinedType{}; }
```

1. 基础类型转`Value`可以直接套用模板，而不需要一个一个去写：

```cpp
template <typename Val>
  requires std::constructible_from<value_t, Val &&>
explicit Value(Val &&val) : value_(std::forward<Val>(val))
{}
```

1. `AttrType`不需要额外开一个成员变量存储，直接`AttrType attr_type() const { return AttrType(value_.index()); }`即可。

此外，原本`Value`只支持字符串，后续需要支持向量类型，那么其中的`length()`就带来了歧义。它的含义类似`sizeof`，而不是`std::string`/`std::vector<T>`中的`size()`，可以考虑将其换成`bytes()`来明确含义。

当然，今年参加CCF数据库暑期学校的同学也可以考虑用`std::variant`去重构一下`Column`类，去年我和同学参加的时候，遇到了不少手动管理内存导致的BUG。

#### 空值

实现空值时通常会面临两个问题：

* **空值的类型归属**：空值是某个类型的特殊值（例如 `INT` 类型中的 `NULL`），还是作为独立类型存在？
* **行存中的存储方式**：空值应如何在内存中表示与存储？

针对第一个问题，可以参考 C++ 中的 `std::nullptr_t`。空值应被视为一种**独立的类型**，而非各个类型的特殊值。空值本身不能转换为其他类型，但任何类型的值都可以转换为空值。在计算、比较和索引操作中，需要对空值进行特殊处理，以保证语义正确。

对于第二个问题，行存布局中通常通过**位图（bitmap）**来记录空值状态（可以参考 `common/lang/bitmap.h` 中的封装）。另一种方式是在每列前预留一个字节，用于标记该字段是否为空。无论采用哪种方式，都可以在保证存储紧凑性的同时，高效判断字段是否为空。

#### 比较和哈希

在数据库系统中，空值在不同场景下的处理方式有所不同，这直接影响比较表达式、排序、索引和聚合的实现逻辑。在当前 `Value` 实现中，空值由 `std::nullptr_t` 表示，可以通过 `Value::is_null()` 判断。

* **比较表达式**
  在计算普通比较表达式时，空值与空值或其他非空值无法判断大小，也不能直接判定相等。即对于表达式 `Value::compare(other)`，若任一值为空，通常返回非真值（`NULL` 或异常标识），以符合 SQL 的三值逻辑。

* **排序**
  在排序操作中，空值通常被视作**比所有非空值都小**，确保排序结果中空值位于最前端。当前实现中，可以通过自定义 `<=>` 运算符处理空值与非空值的比较，保证排序的一致性：

```cpp
inline std::strong_ordering operator<=>(const Value &left, const Value &right)
{
    if (left.is_null() && right.is_null()) { return std::strong_ordering::equal; }
    if (left.is_null()) { return std::strong_ordering::less; }
    if (right.is_null()) { return std::strong_ordering::greater; }
    const int cmp = left.compare(right);
    return (cmp < 0 ? std::strong_ordering::less : cmp > 0 ? std::strong_ordering::greater : std::strong_ordering::equal);
}
```

* **索引和聚合**
  在一般索引和聚合操作中，空值与空值应被视作相等，以便正确删除或聚合对应行。这里可以复用排序所用到的比较函数。
  对于唯一索引，空值不受唯一约束，可以存在多行空值。这要求在索引构建和唯一性检查时对空值进行特判。

* **哈希计算**
  空值的哈希应保持一致性。当前 `Value` 实现中，空值对应的哈希值固定为 0，以保证在哈希表或哈希索引中空值可正确定位。

总结来看，空值在不同场景下的行为可概括为：

* **索引与聚合**：空值与空值相等
* **排序**：空值小于所有非空值
* **比较表达式**：空值与任意值比较返回非真
* **唯一约束**：空值不受唯一性限制，需要特殊处理

### Expression

在数据库执行框架中，`Expression` 是一个高度抽象的类型，用于统一表示各种可计算的值或逻辑表达式。然而，当前 MiniOB 代码中，这种抽象并未得到充分应用。

例如，在 SQL 解析阶段：

* `WHERE` 子句理应生成一个 `Expression` 对象，但在当前实现中，`sql/parser/yacc_sql.y` 中仅使用 `condition_list` 来表示。
* `condition_list` 只支持 `AND` 连接，并且其中的 `condition` 并非真正的 `Expression`。虽然后续阶段会将其转换为 `Expression`，但这一转换发生在解析之后，导致早期阶段缺少统一的表达式抽象。

类似情况还出现在 `INSERT` 和 `UPDATE` 的值处理上，当前的 `value` 对象也可以被抽象为 `Expression`，从而统一解析和计算逻辑。这对于复杂表达式（例如嵌套函数、子查询、算术运算等）来说，后续的扩展十分困难。因此，应该尽早将此处重构，后续扩展功能反而会变得简单。

### Subquery

子查询通常是 SQL 执行中最复杂的部分之一，涉及内外层查询的联动、作用域管理以及表达式求值顺序。

在当前 MiniOB 比赛环境下，子查询的性能要求不高，因此可以先实现最基础的功能，但在接口设计阶段仍需提前考虑复杂子查询的支持，否则后续扩展和修改将非常困难。

**关键考虑点**：

* **内外层联动**
  子查询可能引用外层查询的列，接口设计应支持内层查询访问外层变量，并在执行阶段正确绑定。

* **统一表达式抽象**
  子查询通常作为表达式的一部分出现（例如 `WHERE` 条件、`SELECT` 列或聚合函数参数），接口应允许将子查询包装为 `Expression` 对象，以便与普通值和计算表达式统一处理。

* **可扩展性**
  即便初期实现只支持基础子查询，也应保证接口能够支持：

  * 嵌套多层子查询
  * EXISTS / IN / ANY 等多种语法
  * 子查询与聚合函数、排序或限制条件结合

通过提前规划这些接口，可以确保即使初始实现简单，也能够顺利扩展到更复杂的子查询场景，减少未来重构的成本。

==TODO==

#### 多返回值expression接口设计

#### SQL各处理阶段的封装

#### 内外层联动如何传参（`thread_local`的妙用）

#### 注意事项

* 需要先返回SQL是否执行成功，再返回结果
* conjunction simplification rule存在bug，

### Alias和只读View

### 多列索引

### 唯一索引、MVCC和事务

### TEXT和大VECTOR

### Functions

### 可修改View

### ALTER TABLE

### UNION

### 外部排序

### 向量索引和全文索引

## 复赛浅谈

今年复赛包含两道题，分别是优化带条件过滤的全文索引性能和提升 RAG 准确率，最终经过不同路径的优化汇聚，团队获得第四名，并斩获二等奖，归一化成绩如下（本人主要负责第一题）：

![复赛成绩](seekdb-score.png)

今年复赛颇具挑战，第一题性能优化主要依赖火焰图定位瓶颈，但部分大型优化（如迭代器重构）及一些细微优化未完成，整体表现略逊前三名；第二题由队友完成，通过关键词生成和加权排序实现，方法简洁且效果稳定优异。
