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

- OceanBase 数据库大赛：行存 + 火山模型 + 功能实现
- CCF 数据库暑期学校：PAX 存储 + 向量化执行模型 + OLAP 性能优化
- 部分高校课程作业：LSM 树存储引擎

但近几年 OceanBase 数据库大赛仅使用了**行存格式 + 火山执行模型**这一路径，其它模块（如向量化执行引擎、LSM 树相关代码）在比赛中完全不会被用到。
因此，从工程简化的角度来看，参赛者甚至可以在比赛开始前直接移除这些无关模块，以减少理解成本，也有助于聚焦真正与赛题相关的核心逻辑。

## 开发机配置

如果你不幸和我一样，服务器的系统是CentOS7，那么建议你新开一个账户并使用conda：

```bash
curl https://repo.anaconda.com/miniconda/Miniconda3-py312_25.7.0-2-Linux-x86_64.sh > Miniconda.sh
bash Miniconda.sh -p ~/conda
```

需要配置环境变量：

```bash
export CONDA_PREFIX=$HOME/conda
export CPATH=$CONDA_PREFIX/include:$CPATH
export LIBRARY_PATH=$CONDA_PREFIX/lib:$LIBRARY_PATH
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
export C_INCLUDE_PATH=$CONDA_PREFIX/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$CONDA_PREFIX/include:$CPLUS_INCLUDE_PATH
```

需要安装高版本gcc等：

```bash
conda install -c conda-forge gcc=13.2.0 gxx=13.2.0
```

如果编译报错说找不到`thread`那可能并不是找不到，而是需要安装`compiler-rt`。

vscode需要版本号小于等于`1.85.2`，或者使用[vscode-server-centos7](https://github.com/MikeWang000000/vscode-server-centos7)

如果想在vscode使用LLDB调试请安装插件`lldb-dap`，`launch.json`新增配置：

```json
{
     "type": "lldb-dap",
     "request": "launch",
     "name": "Launch(LLDB-DAP)",
     "program": "${workspaceFolder}/build/bin/observer",
     "args": ["-f", "${workspaceFolder}/etc/observer.ini", "-P", "cli"],
     "cwd": "${workspaceFolder}/build",
     "runInTerminal": true
}
```

## 设计思路

### 类型系统

#### `Value`类

MiniOB 中用于存放值的 `Value` 类型基于 `union` 实现，并采用了**手动内存管理**的方式。这种设计在早期 C++ 中较为常见，但随着代码规模的增长，其复杂性和维护成本也逐渐显现出来。

从功能实现上看，当前 `Value` 的逻辑还存在一定程度的**冗余与重复**。例如：

- 类型转换主要集中在 `cast_to` 中实现；
- 但在 `get_类型` 系列函数中，又再次内嵌了一套转换逻辑；
- 实际上，在调用这些 `get_类型` 函数之前，绝大多数场景都会先进行类型检查，因此这里的二次转换几乎永远不会生效，反而成为**冗余分支**，带来额外的分支预测开销；
- 更不一致的是，在 `Value::get_string_t()` 中又完全不做类型转换，进一步加剧了接口语义的不统一。

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
Date                 get_date() const { return std::get<Date>(value_); }
const string        &get_string() const { return std::get<string>(value_); }
string              &get_string() { return std::get<string>(value_); }
const vector<float> &get_vector() const { return std::get<vector<float>>(value_); }

void set_null() { value_ = nullptr; }
void set_int(int val) { value_ = val; }
void set_float(float val) { value_ = val; }
void set_date(Date val) { value_ = val; }
void set_string(string &&val) { value_ = std::move(val); }
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

- **空值的类型归属**：空值是某个类型的特殊值（例如 `INT` 类型中的 `NULL`），还是作为独立类型存在？
- **行存中的存储方式**：空值应如何在内存中表示与存储？

针对第一个问题，可以参考 C++ 中的 `std::nullptr_t`。空值应被视为一种**独立的类型**，而非各个类型的特殊值。空值本身不能转换为其他类型，但任何类型的值都可以转换为空值。在计算、比较和索引操作中，需要对空值进行特殊处理，以保证语义正确。

对于第二个问题，行存布局中通常通过**位图（bitmap）**来记录空值状态（可以参考 `common/lang/bitmap.h` 中的封装）。另一种方式是在每列前预留一个字节，用于标记该字段是否为空。无论采用哪种方式，都可以在保证存储紧凑性的同时，高效判断字段是否为空。

#### 比较和哈希

在数据库系统中，空值在不同场景下的处理方式有所不同，这直接影响比较表达式、排序、索引和聚合的实现逻辑。在当前 `Value` 实现中，空值由 `std::nullptr_t` 表示，可以通过 `Value::is_null()` 判断。

- **比较表达式**
  在计算普通比较表达式时，空值与空值或其他非空值无法判断大小，也不能直接判定相等。即对于表达式 `Value::compare(other)`，若任一值为空，通常返回非真值（`NULL` 或异常标识），以符合 SQL 的三值逻辑。

- **排序**
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

- **索引和聚合**
  在一般索引和聚合操作中，空值与空值应被视作相等，以便正确删除或聚合对应行。这里可以复用排序所用到的比较函数。
  对于唯一索引，空值不受唯一约束，可以存在多行空值。这要求在索引构建和唯一性检查时对空值进行特判。

- **哈希计算**
  空值的哈希应保持一致性。当前 `Value` 实现中，空值对应的哈希值固定为 0，以保证在哈希表或哈希索引中空值可正确定位。

总结来看，空值在不同场景下的行为可概括为：

- **索引与聚合**：空值与空值相等
- **排序**：空值小于所有非空值
- **比较表达式**：空值与任意值比较返回非真
- **唯一约束**：空值不受唯一性限制，需要特殊处理

### Expression

在数据库执行框架中，`Expression` 是一个高度抽象的类型，用于统一表示各种可计算的值或逻辑表达式。然而，当前 MiniOB 代码中，这种抽象并未得到充分应用。

例如，在 SQL 解析阶段：

- `WHERE` 子句理应生成一个 `Expression` 对象，但在当前实现中，`sql/parser/yacc_sql.y` 中仅使用 `condition_list` 来表示。
- `condition_list` 只支持 `AND` 连接，并且其中的 `condition` 并非真正的 `Expression`。虽然后续阶段会将其转换为 `Expression`，但这一转换发生在解析之后，导致早期阶段缺少统一的表达式抽象。

类似情况还出现在 `INSERT` 和 `UPDATE` 的值处理上，当前的 `value` 对象也可以被抽象为 `Expression`，从而统一解析和计算逻辑。这对于复杂表达式（例如嵌套函数、子查询、算术运算等）来说，后续的扩展十分困难。因此，应该尽早将此处重构，后续扩展功能反而会变得简单。

### Subquery

子查询通常是 SQL 执行中最复杂的部分之一，涉及内外层查询的联动、作用域管理以及表达式求值顺序。

在当前 MiniOB 比赛环境下，子查询的性能要求不高，因此可以先实现最基础的功能，但在接口设计阶段仍需提前考虑复杂子查询的支持，否则后续扩展和修改将非常困难。

**关键考虑点**：

- **内外层联动**
  子查询可能引用外层查询的列，接口设计应支持内层查询访问外层变量，并在执行阶段正确绑定。

- **统一表达式抽象**
  子查询通常作为表达式的一部分出现（例如 `WHERE` 条件、`SELECT` 列或聚合函数参数），接口应允许将子查询包装为 `Expression` 对象，以便与普通值和计算表达式统一处理。

- **可扩展性**
  即便初期实现只支持基础子查询，也应保证接口能够支持：

  - 嵌套多层子查询
  - EXISTS / IN / ANY 等多种语法
  - 子查询与聚合函数、排序或限制条件结合

通过提前规划这些接口，可以确保即使初始实现简单，也能够顺利扩展到更复杂的子查询场景，减少未来重构的成本。

#### SQL各处理阶段的封装

在MiniOB中，SQL的处理分为：

- parse (String $\rarr$ SQL Node)
- resolve (SQL Node $\rarr$ Stmt)
- generate logical operator (Stmt $\rarr$ Logical Operator)
- generate physical operator (Logical Operator $\rarr$ Physical Operator)
- execute

各阶段的处理可以做一下封装。它在子查询相关的所有功能实现中都有可能复用(后续的Alias和View那些)。

#### 内外层联动如何传参（`thread_local`的妙用）

在子查询的执行逻辑中，我们面临一个典型的接口不匹配难题：`get_value()` 或 `try_get_value()` 在调用时，无法直接向下透传 `PhysicalOperator::open()` 所必需的 `Trx *trx` 指针；与此同时，`PhysicalOperator` 在实际执行过程中，也无法接收由 `get_value()` 传入的 `Tuple *tuple` 作为入参。

按照常规的重构思路，应当修改 `Expression` 与 `PhysicalOperator` 的底层接口，显式增加这两个参数。然而，这类底层接口的改动无异于“动全身”的大手术，涉及的代码量和潜在风险极大。

##### 借力全局变量的思路

在单线程环境（尤其是 C 语言编程）中，全局变量是实现跨层级数据传递的常用手段。为了绕过繁琐的接口修改，我们可以考虑将 `Trx *trx` 和 `Tuple *tuple` 存放在全局作用域中，使其在任何执行路径下都“触手可及”。

然而，数据库是一个高度并发的多线程系统，传统的全局变量会引发严重的线程安全问题。

##### Thread-per-Connection 模型与 thread_local

幸运的是，大多数 OLTP 数据库系统遵循 **Thread-per-Connection**（或 **One-Thread-per-Connection**）模型。在该模型下，单个线程在执行周期内通常绑定一个特定的事务，事务与线程具有明确的一一对应关系。

基于这一特性，我们可以使用 `thread_local` 代替全局变量。这样既保留了全局访问的便利性，又实现了线程间的隔离，确保每个执行线程只能访问到属于自己的事务上下文。

##### 利用 RAII 实现参数压栈

针对内外层联动的子查询，我们可以借鉴函数调用的栈帧思想。当执行进入 `get_value()` 时，将当前 `Tuple` 压入线程局部的上下文栈中；当执行结束返回时，再将其弹出。

为了确保栈操作的安全性和对称性（防止因异常导致未能正确弹出），推荐使用 **RAII** 机制进行管理。我们可以实现一个名为 `TupleContextGuard` 的类来自动化这一过程：

```cpp
struct TupleContextGuard {
  TupleContextGuard(const Tuple *tuple) { ThreadLocalContext::current().push_tuple(tuple); }
  ~TupleContextGuard() { ThreadLocalContext::current().pop_tuple(); }
};

```

##### thread_local 的更多延伸应用

除了处理子查询传参，`thread_local` 在数据库开发中还有许多进阶用途：

- **局部性能缓存：两级缓存设计** 在数据库中，元数据（如 Table Schema）通常存储在全局的 HashMap 中。为了避免高并发下对全局锁的频繁争夺，我们可以利用 thread_local 维护一个“影子缓存”:

```cpp
std::unordered_map<std::string, TableMeta> g_metadata_manager;
std::shared_mutex g_meta_mutex;

TableMeta get_table_meta(const std::string& table_name) {
  thread_local std::unordered_map<std::string, TableMeta> local_meta_cache;
  // 1. 尝试从线程本地缓存获取
  auto it = local_meta_cache.find(table_name);
  if (it != local_meta_cache.end()) {
    return it->second; // 命中本地缓存，无锁返回
  }
  // 2. 本地未命中，访问全局并加读锁
  std::shared_lock lock(g_meta_mutex);
  if (g_metadata_manager.contains(table_name)) {
    TableMeta meta = g_metadata_manager[table_name];
    // 3. 回填到本地缓存，下次即可无锁访问
    local_meta_cache[table_name] = meta;
    return meta;
  }
  return TableMeta::NotFound();
}
```

- **指标统计**：在高吞吐量的扫描任务中，如果所有线程都使用 `std::atomic` ，CPU 的缓存行同步（Cache Line Bouncing）会成为严重瓶颈。每个线程维护一套独立的性能监控计数器，在任务结束时再统一汇总，可以消除原子操作带来的性能开销。

- **内存池管理**：为每个线程分配独立的线程本地内存池（Thread-Local Memory Pool），大幅提升高并发下的内存分配效率。

#### 注意事项

**1. 协议状态预判与执行期错误的冲突**

在当前的测试框架中，系统对 SQL 的响应遵循固定的交互序列：必须先回传执行状态（`SUCCESS`/`FAILURE`），随后才开始返回结果集。按照原有的逻辑，该状态位在 `ResultSet::open` 阶段即被锁定（具体可见 `PlainCommunicator::write_result_internal` 的实现逻辑）。

然而，引入子查询机制后，这一逻辑面临挑战。由于子查询的嵌套执行特性，部分复杂的查询在 `open` 阶段往往无法预知后续执行逻辑是否会触发异常。

此时产生了一个严重的副作用：如果系统在 `open` 时预先返回了 `SUCCESS` 状态，而在后续流式获取数据的过程中发生运行时报错，将会导致协议层面的前后矛盾。这种状态违约会直接导致客户端连接异常断开。

一般来讲，有两种解决方案:

1. `ResultSet::open`阶段将结果集缓存；
2. `ResultSet::open`先将SQL运行一遍获取执行状态。

一般会选择实现第一种。在2025年的初赛中，对运行内存做了限制，对于`big-order-by`这种内存中放不下整个结果集的查询，可以考虑手动设定一个阈值，即只缓存前`N`行结果。

**2. conjunction simplification rule存在bug**

[#440](https://github.com/oceanbase/miniob/issues/440)
[#637](https://github.com/oceanbase/miniob/issues/637)
可以看下我的一个[简单修复](https://github.com/what-is-me/miniob-fork/commit/ace50f9498b919cf52cdf111faf44225cb50c8a3)

**3. 其他方法**

往年有老哥提到，内外联动的复杂子查询可以在优化阶段转成`JOIN`，这样就没有传参问题了~~，但实现起来比较复杂，容易出BUG，如果没有性能需求的话其实不建议搞~~。

### Alias 与只读 View

在完成了子查询的基础逻辑后，Alias（表别名）与只读 View（视图）的实现便水到渠成了。从本质上讲，它们都可以被视为一种“虚拟表”或存储好的“子查询”。

在架构设计上，许多成熟队伍会选择引入 `TableBase` 基类，让物理表 `Table` 和视图 `View` 共同继承。但在训练营的轻量级框架下，为了追求实现简便，我们采取了一种更讨巧的方案：**直接复用 `Table` 类，并利用 ID 的正负来区分身份。**

#### 巧妙的 ID 映射方案

由于数据库中真实物理表的 `table_id` 均为正数，我们可以约定：**凡是 ID 为负数的 `Table` 对象，均视为视图或子查询生成的临时表。**

在 `Table` 类中，我们通过重载构造函数来实现这一逻辑：

```cpp
// 视图/子查询构造函数：通过负数 view_id 赋予其“虚拟”身份
Table(const string &name, vector<FieldMeta> &&field_metas, int view_id)
    : table_meta_(name, std::move(field_metas), view_id) {}

// Alias 构造函数：拷贝原始表的元数据，仅修改表名
Table(const string &name, Table &table) 
    : db_(table.db_), table_meta_(name, table.table_meta_), engine_(table.engine_) {}

[[nodiscard]] bool is_view() const { return table_id() < 0; }

```

#### `ThreadLocalContext`：虚拟表的“临时集散地”

视图和别名通常只在当前会话或当前 SQL 执行周期内有效，并且在resove阶段，`Table`都是以裸指针的形式出现，因此 `ThreadLocalContext` 成为了管理这些动态对象的绝佳容器。它开辟了一个数组来维护这些临时 `Table` 实例的生命周期，并通过映射表将作为视图的 `Table` 对象与其对应的逻辑算子树关联起来。

在 `ThreadLocalContext` 的实现中，几个核心方法串联起了整个流程：

- **`alias()`**：为现有表创建一个“马甲”，复用引擎和数据库指针，但拥有独立的别名。
- **`subquery()`** & **`view()`**：两者最终都会汇聚到内部私有方法 `logical_operator()`。该方法会根据逻辑算子树推导出 Schema（`FieldMeta`），并生成一个具有负数 ID 的 `Table` 实例。
- **`set_subquery_tag()`**：这是一个细节优化。它会递归向下寻找 `PROJECTION` 算子，并将别名注入其中，确保在执行期进行列解析时，能够正确识别出这些来自子查询或视图的字段。

#### 内存与生命周期管理

由于 `ThreadLocalContext` 的生命周期较长，我们需要在每条 SQL 执行结束后（或 Session 切换时）及时清理，防止内存泄漏或上下文污染：

```cpp
void ThreadLocalContext::clear() {
  session_ = nullptr;
  // 清空复合 Tuple 栈
  static_cast<CompositeViewTuple *>(tuple_.get())->tuples().clear();
  // 销毁所有临时创建的 Table 定义
  table_def_holder_.clear();
  // 释放关联的逻辑算子树
  views_.clear();
}

```

通过这种方式，我们利用 `thread_local` 在不破坏原有存储引擎接口的前提下，灵活地支持了 Alias、子查询和视图，实现了代码的高度复用。

### 多列索引

MiniOB 的早期实现仅支持非空的单列索引，其接口设计高度依赖原始字节指针（`const char *`）。这种设计在处理复合索引、存在 `NULL` 值的场景或变长字段时，往往会导致逻辑碎片化且难以维护。

#### 接口的现代化演进

为了支持复合索引并提升代码的健壮性，索引层的核心接口被重构为基于 `span<const Value>` 的形式。这一改变不仅增强了类型安全性，还让多列数据的传递变得直观且统一。

重构后的接口如下：

```cpp
// 现代化索引接口定义
RC insert_entry(span<const Value> key, const RID &rid);
RC delete_entry(span<const Value> key, const RID &rid);
IndexScanner *create_scanner(
  span<const Value> left_key, bool left_inclusive, 
  span<const Value> right_key, bool right_inclusive
);

```

#### 核心工具：TupleEncoder 的实现

在将多列数据持久化到 B+ 树或其他索引结构时，必须将逻辑上的 `vector<Value>` 序列化为连续的字节流。为此，应该实现一个 `TupleEncoder` 工具类，用于处理元组与 `char*` 之间的相互转换。

该工具类不仅服务于索引系统，在**外部排序（External Sort）**等需要频繁对元组进行序列化和反序列化的场景中也具有极高的通用性。

`TupleEncoder` 的设计要点包括：

- **内存布局预计算**：在构造阶段预先计算每个字段的 `offsets_` 和总长度 `total_length_`。元组的起始位置预留了一段 **Bitmap（位图）** 空间，用于标记哪些列为 `NULL`。
- **高效编码 (`encode`)**：遍历输入的 `Value` 向量，根据字段元数据将其原始数据拷贝到对应的偏移位置，并同步更新头部的 Null Bitmap。
- **精准解码 (`decode`)**：通过位图判断字段有效性，并结合偏移量与类型信息重新构建 `Value` 对象，实现数据的零损还原。

这里给出`encode`的实现：

```cpp
class TupleEncoder {
public:
  // 支持从 FieldMeta 向量或原始类型数组构造
  explicit TupleEncoder(span<FieldMeta> fields);

  [[nodiscard]] vector<char> encode(span<const Value> tuple) const {
    vector<char> res(total_length_);
    // 在序列化数据的头部维护位图，解决 NULL 值存储问题
    common::Bitmap nulls(res.data(), attr_types_.size()); 
    
    for (int i = 0; i < tuple.size(); ++i) {
      if (tuple[i].is_null()) {
        nulls.set_bit(i);
      } else {
        nulls.clear_bit(i);
        // 按预分配的偏移量精准填充内存
        std::memcpy(res.data() + offsets_[i], tuple[i].data(), 
                    std::min(attr_lengths_[i], tuple[i].bytes()));
      }
    }
    return res;
  }
};
```

#### 基于 `TupleEncoder` 的通用比较器

在实现了 `TupleEncoder` 之后，`AttrComparator` 就比较好写了，并且之前实现的 `Value` 的 `<=>`，也能在此处简化代码。

```cpp
class AttrComparator
{
public:
  void init(span<FieldMeta> fields) { encoder_ = TupleEncoder(fields); }
  [[nodiscard]] int key_length() const { return encoder_.total_length(); }
  std::strong_ordering operator()(const char *v1, const char *v2) const
  {
    auto left_list  = encoder_.decode(v1);
    auto right_list = encoder_.decode(v2);
    return left_list <=> right_list;
  }

private:
  TupleEncoder encoder_;
};
```

很显然，这里有不少复制开销，如果后续有性能需求可以在此基础上做点优化。

### 唯一索引、MVCC 与事务回滚

在数据库内核开发中，索引不仅仅是加速查询的工具，更是约束（Constraint）的守护者。当引入**唯一索引（Unique Index）**和 **MVCC（多版本并发控制）**后，数据的插入与更新逻辑变得复杂。即使在模拟串行执行的训练营测试中，为了保证 ACID 特性中的原子性（Atomicity），一套完备的事务机制也是必不可少的。

#### 唯一索引

在 `Index::insert_entry` 的实现中，唯一性检查不再是简单的“查查 B+ 树里有没有这个 Key”。

1. **非空约束参与**：根据 SQL 标准，多个 `NULL` 值通常不被视为重复。因此，校验逻辑首先要确认 Key 序列中是否存在空值。
2. **多版本下的可见性**：

- 在普通的 **VacuousTrx**模型下，只需调用索引引擎的 `exists` 接口判断物理存在即可。

- 但在 **MVCC 模式**下，物理存在的 Key 对应的记录可能已经被标记删除，或者是由尚未提交的事务所创建，需要仔细的判断。

#### 多行 `UPDATE` 的错误处理

在处理 `UPDATE` 语句时，即使没有并发冲突，操作也可能因为各种原因失败（例如修改后的值违反了唯一约束）。如果一条 `UPDATE` 语句需要修改 10 行数据，在修改到第 5 行时报错，那么前 4 行已经变更的物理数据必须能够回滚到初始状态。

为此，可以为 `VacuousTrx` 设计一个轻量级的 **UndoLog**：

- **逻辑日志**：记录操作类型（`INSERT_RECORD` / `DELETE_RECORD`）、目标表以及 Record 的快照。
- **深拷贝**：注意 `Record` 类在实现时可能仅持有原始数据的指针（浅拷贝）。为了确保回滚时数据依然有效，需要实现 `deep_copy()` 函数，将元组数据真正持久化到 Log 中，避免回滚时引用了已经失效的内存。
- **逆序回滚**：当发生错误时，UndoLog 按照**先进后出（FILO）**的顺序进行补偿操作（Delete 补偿 Insert，Insert 补偿 Delete），确保数据库状态的一致性。

#### 多行 `UPDATE` 的执行顺序

为了处理复杂的更新逻辑，推荐将 `UPDATE` 拆解为三个阶段：

1. **Select**：先扫描出所有符合条件的 `RID` 及其修改后的数据。
2. **Delete**：执行物理删除。
3. **Insert**：执行物理插入。

**案例分析：**
假设表 `t1` 只有一列 `id (int, not null)`，包含两行数据：`1` 和 `2`。执行 `UPDATE t1 SET id = 3 - id`。

- **如果逐行更新**：处理第一行 `1 -> 2` 时，会因为唯一索引中已存在 `2` 而报错。
- **标准姿势**：

1. 扫描发现：`RID1(1)` 准备变为 `2`，`RID2(2)` 准备变为 `1`。
2. 删除 `RID1` 和 `RID2`。
3. 插入 `2` 和 `1`。
整个过程顺利完成。

**注意** 2025年新增了一个测例：

![image-20260126142820551](image-20260126142820551.png)

~~但是因为mvcc-update是唯一一题数据不随机的题目，所以真可以打表过（笑）。~~

### TEXT和大VECTOR

在处理 `TEXT`（最大可达 `CHAR(65535)`）或高维 `VECTOR` 类型时，数据库引擎面临的最直接问题是**单页面容量限制**。如果直接将这些大数据类型放置在普通的 Data Page 中，会导致行溢出甚至页面无法容纳单条记录。

在往年的训练营中，部分队伍通过简单粗暴地增大 `PAGE_SIZE` 来通过测试，但这在 2025 年的评测环境中往往会触发 **OOM** 报错。这是因为大页面会显著增加 Buffer Pool 的内存开销和 I/O 碎片。

为了以最小的代码改动量解决这一问题，可以采用**溢出页（Overflow Page）**或**外部文件存储**的思路：

- **阈值判定**：为字段设定一个长度阈值（例如 1KB）。
- **非在页存储（Off-page Storage）**：如果字段的最大定义长度超过该阈值，不论其实际存储的数据长短，都将其内容写入独立的外部文件或专用的溢出页面中。
- **引用占位**：在原始的 Data Page 中，仅保留一个固定长度的“指针”，包含数据的**实际长度**和在外部文件中的**偏移量（Offset）**。

这种方案的优势在于不需要修改建表逻辑和底层的元数据存储结构，只需在 `Record` 的 `serialize` 和 `deserialize` 环节增加一层路由逻辑。

虽然理想的做法是在建表（`CREATE TABLE`）阶段就精细化定义每个字段的存储属性（Storage Class），但这涉及到 SQL 解析器、逻辑计划以及物理引擎的深度修改。对于现阶段的开发，采用基于长度阈值的外部存储 + 引用占位方案是投入产出比最高的选择。

### Functions

在数据库内核设计中，内置函数（Built-in Functions）的实现非常适合采用单例工厂模式。通过建立一个全局哈希表作为注册中心，可以在程序启动阶段将所有支持的函数（如数学运算、字符串处理、向量计算等）动态注入系统。哈希表的键通常采用“函数名”或“函数名 + 参数特征（Overload）”的组合，这种设计在处理函数别名（Alias）或批量注册相似逻辑（如不同类型的显式转换函数）时表现出极高的灵活性。

有些实现方案会选择将函数名作为保留字直接写进 Yacc（解析器生成器） 的语法规则中。虽然这种做法在处理简单的固定逻辑时能奏效，但在工业级实现中通常是不可取的。首先，这会导致语法文件极其臃肿，每增加一个函数都需要重新编译解析器，严重违反了开闭原则；其次，解析层不应关注具体的计算逻辑，将函数名视为普通的标识符（Identifier）并在绑定（Binding）阶段从工厂中动态检索，能够实现解析层与执行层的完美解耦。此外，动态注册机制还为后续支持 UDF（用户自定义函数） 预留了接口，这是硬编码在语法中的方案所无法比拟的优势。

### 可修改View

在支持了基础视图后，一个进阶的挑战是如何让视图“动起来”。并非所有视图都支持增删改，其核心原则在于：**数据库必须能明确地将视图操作映射回基表（Base Table）的唯一物理行。**具体规则见[视图更新规则说明](https://oceanbase.github.io/miniob/game/create-view-2025/)。

为了管理不同性质的表，建议在 `Db` 类中维护三个哈希表，分别存放物理表、只读视图和可修改视图。通过这种分类，可以在解析阶段快速判定操作的合法性。

这里给出我实现的可修改视图的结构：

```cpp
class WriteableView
{
public:
  struct ProjectedTable
  {
    Table      *table_;
    vector<int> cols_;
  };
  RC init(
      Db *db, const string &view_name, span<const unique_ptr<Expression>> expressions, const vector<Table *> &tables);
  [[nodiscard]] int           get_id() const { return view_id_; }
  void                        set_id(int id) { view_id_ = id; }
  [[nodiscard]] const string &name() const { return view_name_; }
  // full_insert 相关：insert into view_name values(...)
  [[nodiscard]] std::optional<ProjectedTable> get_full_insert_table() const;
  // partial_insert 和 update 相关：insert into view_name(a,b) values(...); update view_name set a=...;
  [[nodiscard]] std::optional<ProjectedTable> get_projected_table(vector<string> attr_list) const;
  // partial_insert 相关
  [[nodiscard]] bool support_insert() const { return support_insert_; }
  // delete 相关：delete from view_name where ...
  [[nodiscard]] Table *get_deletable_table() const;

private:
  [[nodiscard]] std::optional<ProjectedTable> get_full_insert_table_impl() const;
  [[nodiscard]] std::optional<ProjectedTable> get_projected_table_impl(vector<string> attr_list) const;

  void   collect_table_ids(vector<int> &table_ids) const;
  void   collect_tables_self_join();
  Db    *db_;
  string view_name_;
  int    view_id_;

  vector<unique_ptr<Table>>            table_schemas_;
  vector<unique_ptr<Field>>            fields_;
  unordered_map<string, const Field *> field_alias_;
  unordered_set<int>                   tables_unable_modify_;

  bool support_full_insert_{true};
  bool support_insert_{true};
  bool support_delete_{true};
};
```

#### 核心架构：映射与重定向

实现可修改视图的关键在于建立 **View Column 到 Base Table Column 的映射链路**。在代码实现中，定义了 `ProjectedTable` 结构体，它记录了目标物理表及其对应的列 ID 序列（`cols_`）。

当执行 `INSERT` 或 `UPDATE` 时，系统不再直接操作视图，而是通过 `get_projected_table` 或 `get_full_insert_table` 找到底层的 `ProjectedTable`，将操作“重定向”给物理表。

#### 判定逻辑与约束检查

在 `WriteableView::init` 过程中，会进行严苛的合法性审查，以决定视图的“可写边界”：

1. **单表限制**：对于 `DELETE` 和全列 `INSERT`（`full_insert`），视图必须基于单表。如果是多表 Join 生成的视图，虽然可能支持特定列的 `UPDATE`，但通常禁止删除和全列插入，因为这会产生歧义。
2. **表达式过滤**：如果视图包含计算列（如 `id + 1`），该列会被标记为不可写。如果全列插入涉及此类列，整个 `support_full_insert_` 标记将被置为 `false`。
3. **Self-Join 冲突**：通过 `collect_tables_self_join` 检测视图是否在底层多次引用了同一张物理表。如果存在自连接，为了避免更新时的逻辑混乱，通常会将相关表标记为不可修改（`tables_unable_modify_`）。
4. **聚合检测**：利用 `has_aggr_expr` 递归检查 `SELECT` 列表、`GROUP BY` 或 `ORDER BY` 中是否含有聚合函数。一旦发现聚合逻辑，该视图将自动降级为只读。

#### 嵌套视图的递归映射

当出现“视图上的视图”时，映射逻辑通过递归调用实现。例如在 `get_full_insert_table_impl` 中：

```cpp
if (table->is_view()) {
  auto real_table = db_->find_writeable_view(table->table_id())->get_full_insert_table().value();
  table           = real_table.table_;
  for (auto &col_id : cols) {
    col_id = real_table.cols_[col_id];
  }
}
```

这种设计确保了无论视图嵌套多少层，最终执行器拿到的都是最底层的物理表引用和正确的偏移量。

#### 细节优化：别名转换

由于视图可能对基表列进行了重命名，`field_alias_` 映射表在 `UPDATE` 或 `partial_insert`（带列名的插入）中起到了翻译官的作用。它将用户可见的别名还原为物理字段名，再通过底层 ID 完成数据落盘。

### ALTER TABLE

- 重命名表名：
  1. 根据表结构创建新表
  2. 复制数据
  3. 删除旧表
  4. 如果1~2失败则删除新表即可
- 增删列/修改列名/修改列属性：
  1. 根据修改后的表结构定义创建临时表
  2. 复制数据到临时表
  3. 删除旧表
  4. 重命名临时表
  5. 如果1~2失败则删除新表即可

需要注意如果列上有索引删除列需要修改索引，比如，删除列`b`，`idx(a,b)`变成`idx(a)`，`idx(b)`则会删除这个索引。

### UNION

在 2025 年的实验题目中，`UNION` 的实现相对克制，并未引入复杂的嵌套语法（如整个 `UNION` 结果集再套 `ORDER BY`），这有效避免了大规模的底层架构重构。其核心逻辑在于对结果集的合并处理：`UNION` 需要进行去重（通常通过内部哈希表或排序实现），而 `UNION ALL` 则仅需将多个结果集简单地垂直堆叠。

从算子树的视角来看，`UNION` 本质上可以递归地形成一个二叉树结构。然而，在执行效率和逻辑清晰度上，将其**扁平化为多叉树**往往是更优的选择。这种扁平化处理取决于节点类型的连续性：

- **根节点为 UNION 时**：如果一个子树的根节点是 `UNION`（涉及去重），它可以将其下层连续的 `UNION` 操作合并，演化为一个高度为 2 的**多叉树结构**。在这种结构下，所有子查询（Subqueries）的结果集统一输入到一个去重算子中，避免了层层嵌套去重带来的冗余开销。
- **根节点为 UNION ALL 时**：如果根节点是 `UNION ALL`，它会将下层**连续**的 `UNION ALL` 子节点扁平化为一个单一的 **UNION ALL 算子**。
- **混合嵌套场景**：如果 `UNION ALL` 的子节点或子孙节点中出现了 `UNION`，则扁平化会在 `UNION` 节点处“截断”。这是因为 `UNION` 具有去重语义，它必须先作为一个独立的子任务完成内部的合并与去重，然后将其产生的结果集作为整体回传给上层的 `UNION ALL`。

换句话说，`UNION ALL` 可以看作是一个简单的“数据流汇聚器”，它并不关心子节点是普通的 `SELECT` 还是一个复杂的 `UNION` 子树。在逻辑计划生成阶段，我们会尽可能合并同类型的操作符，形成一个宽泛的 Append 算子，直到遇到语义不同的边界（如从 `ALL` 切换到非 `ALL`）。这种从“递归二叉树”到“局部扁平多叉树”的转变，不仅简化了执行逻辑，也极大地提升了结果集处理的吞吐量。

### [外部排序](https://zh.wikipedia.org/wiki/%E5%A4%96%E6%8E%92%E5%BA%8F)

在数据库执行查询时，ORDER BY 是极其耗费资源的算子。当待排序的数据量较小，能够完整装入内存时，可以简单地使用 `std::sort` 等内存排序算法。然而，当数据集规模超出内存预算（如面对 2025 年测试用例中的大数据量场景）时，外部排序（External Merge Sort） 就成了维持系统稳定的唯一出路。

外部排序的核心思想是分治法。其执行过程通常分为两个阶段：生成初始归并段（Run Generation）与多路归并（Multi-way Merge）。

在第一阶段，排序算子从下层算子不断拉取数据，并暂存在内存缓冲区中。为了兼顾效率与通用性，元组会被区分为“排序键（Key）”和“原始行（Row）”两部分。当内存中的数据行数达到预设阈值（例如 400 行）时，系统会对这部分数据进行内部排序，随后利用 TupleEncoder 将其序列化为紧凑的字节流，并持久化到磁盘的临时文件中。这一步将一个巨大的排序任务拆分成了若干个有序的局部“小碎片”。

在第二阶段，系统利用**优先队列（堆）**来实现多路归并。通过创建一个合并迭代器，将所有磁盘上的临时文件和最后剩余的内存缓冲区作为子节点挂载到堆中。归并开始时，每个子节点会贡献出自己“最小”的一行推入堆顶。随后，每当上层调用 next() 获取结果，算子就从堆顶弹出一个全局最小（或最大）的行，并立即从产生该记录的子节点中补入下一行数据。

配合 LobFileHandler 的文件管理与 `TupleEncoder` 的序列化机制，这种“内存缓存 + 磁盘落盘 + 多路归并”的组合确保了磁盘 I/O 的高效与数据的类型安全。这套机制使内核能够从容应对各种复杂的大规模排序挑战，在有限的资源约束下实现确定性的执行效率。

### 向量索引和全文索引

2024年新增了向量相关题目，并要求通过`ann-benchmark`。在训练营中，据说该性能测评依旧是按`debug`模式编译的，改成`release`可以大幅度提升性能，此外可以通过火焰图分析瓶颈来优化：

- MYSQL通信协议存在瓶颈，每个SQL需要初始化一个很大的`std::vector<char>`。实际上只需要分配空间就行，因此将`resize`改成`reserve`就行。
- 另一处较大开销是距离的计算，可以通过1）不开方；2）SIMD加速 来解决。一般服务器上都能用AVX512，但是消费级CPU上可能只能用AVX2。
- `Expr::get_value`根据字符串找列开销也不小，可以用哈希表/复用`Expr::pos_`充当位置缓存来加速。

2024年的复赛好写是优化向量查询性能？倒是和这题相关性很大。

2025年并没有性能测试题目，新增了全文索引相关题目（能不能拆成几题啊），以及RAG题目。

- 全文索引没有性能要求，照着实现即可（开赛时闹了乌龙，分词需要删掉停止词，但计算BM25分数却不需要，第二天偷偷修掉了也不出个公告）。
- RAG并不困难，一个下午搞定了，而且和全文索引没有一点关系，反而需要向量索引（但2025年赛题没有向量索引来着），直接用2024的codebase做就行。（以及我们队过了RAG后第二天测评机把langflow的版本升了，貌似有不少队伍卡在这了）。

这里给一下当时做RAG赛题用的`docker-compose.yaml`：

```yaml
version: "3"

services:
  langflow:
    image: langflowai/langflow:latest
    container_name: ${USER}-langflow
    restart: always
    ports:
      - "${LANGFLOW_PORT:-7860}:7860" # 需要配置LANGFLOW_PORT
    environment:
      OB_DOC_PATH: /oceanbase-doc
      EMBEDDING_NAME: "bge-m3:latest"
      EMBEDDING_BASE_URL: http://172.17.0.1:${OLLAMA_PORT} #需要配置OLLAMA_PORT
      LLM_NAME: qwen
      LLM_API_KEY: ${LLM_API_KEY} # 需要配置LLM_API_KEY
      LLM_BASE_URL: https://dashscope.aliyuncs.com/api/v1
      # QA_SERVER_GET_QUESTION_URL: ""
      # QA_SERVER_POST_ANSWER_URL: ""
      MINIOB_SERVER_SOCKET: /miniob/miniob.sock

    volumes:
      - ./tmp:/miniob
      - ../oceanbase-doc/zh-CN:/oceanbase-doc # 在父文件夹中git clone https://github.com/oceanbase/oceanbase-doc.git
      - langflow_data:/app
volumes:
  langflow_data:
```

2025年的复赛赛题倒是和全文索引以及RAG相关，只不过初赛的RAG和全文索引没啥关系，也不知道有多少队伍给带偏了。

## 复赛浅谈

今年复赛包含两道题，分别是优化带条件过滤的全文索引性能和提升 RAG 准确率，最终经过不同路径的优化汇聚，团队获得第四名，并斩获二等奖，归一化成绩如下（本人主要负责第一题）：

![复赛成绩](seekdb-score.png)

今年复赛颇具挑战，第一题性能优化主要依赖火焰图定位瓶颈，但部分大型优化（如迭代器重构）及一些细微优化未完成，整体表现略逊前三名；第二题由队友完成，通过关键词生成和加权排序实现，方法简洁且效果稳定优异。
