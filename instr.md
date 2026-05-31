# MoonBit 序列化框架 — 比赛项目

## 背景

### 当前 MoonBit 序列化生态现状

| 格式 | 项目 | 是否有 trait 体系 | 是否有 derive | 评估 |
|------|------|------------------|---------------|------|
| JSON | `moonbitlang/core/json` | ✅ `ToJson`/`FromJson` trait | ✅ `derive(ToJson, FromJson)` | 可用，但 derive 参数不稳定，`rename_all`、`repr` 等已被废弃 |
| protobuf | `moonbitlang/protobuf` + `protoc-gen-mbt` | ✅ `Read`/`Write` trait | 通过 .proto 文件生成 | 需外部 Go 工具链 |
| MessagePack | `moonbit-community/msgpack` | ❌ 仅有 `Value` enum | ❌ 无 | 基础编解码，无类型映射 |
| BSON | `ZSeanYves/bsonlite` | ❌ 仅有 `BsonValue` enum | ❌ 无 | 基础编解码，缺半数类型 |
| YAML / TOML / XML | ❌ 完全空白 | ❌ | ❌ | 无 |
| 通用序列化框架（serde 等价物） | ❌ 完全空白 | ❌ | ❌ | **最大空白** |

### 核心缺失：无格式无关的通用序列化框架

每个格式各自为政，没有统一的 `Serialize`/`Deserialize` trait 体系。

---

## 项目目标

构建一个 **serde 风格的通用序列化框架**（纯 MoonBit），包含三部分：

```
用户类型 ──→ derive(Serialize, Deserialize) ──→ Serialize/Deserialize trait
                                                          │
                        ┌─────────────────────────────────┼────────────────────┐
                        ▼                                 ▼                    ▼
                   JSON Serializer                  MsgPack Serializer    BSON Serializer
                   JSON Deserializer                MsgPack Deserializer  BSON Deserializer
```

项目名称建议：`moon-serde`

---

## 架构设计

### 第一层：核心 trait

```moonbit
/// 序列化器接口（格式无关）
pub(open) trait Serializer {
  serialize_bool(Self, Bool)
  serialize_int(Self, Int)
  serialize_string(Self, String)
  serialize_bytes(Self, Bytes)
  serialize_option(Self, Serialize, option: Bool?)
  // 容器
  serialize_seq(Self, len: Int) -> SeqSerializer
  serialize_map(Self, len: Int) -> MapSerializer
  serialize_struct(Self, name: String, fields: Array[(String, &Serialize)])
}

/// 反序列化器接口（格式无关）
pub(open) trait Deserializer {
  deserialize_bool(Self) -> Bool
  deserialize_int(Self) -> Int
  deserialize_string(Self) -> String
  deserialize_bytes(Self) -> Bytes
  // 容器
  deserialize_seq(Self) -> SeqDeserializer
  deserialize_map(Self) -> MapDeserializer
  deserialize_struct(Self, name: String, fields: Array[String]) -> StructDeserializer
}

/// 可序列化标记 trait（类似 serde 的 Serialize）
pub(open) trait Serialize {
  serialize(Self, Serializer)
}

/// 可反序列化标记 trait（类似 serde 的 Deserialize）
pub(open) trait Deserialize {
  deserialize(Deserializer) -> Self
}
```

### 第二层：derive 宏

```moonbit
// 自动派生
struct User {
  name : String
  age : Int
  email : String?
} derive(Serialize, Deserialize)

// 支持字段重命名等配置
struct Config {
  #[serde(rename="db_host")]
  db_host : String
  #[serde(skip_serializing_if="Option::is_none")]
  optional_field : String?
} derive(Serialize, Deserialize)
```

### 第三层：格式后端

| 后端 | 优先级 | 说明 |
|------|--------|------|
| JSON | P0 | 基于现有 `@json` 做适配器，或重新实现更完整的 JSON 编解码 |
| MessagePack | P0 | 完整实现 msgpack spec，作为二进制格式代表 |
| BSON | P1 | 基于 msgpack 后端的架构扩展 |
| YAML | P2 | 可选 |

---

## 工作量估算与模块划分

| 模块 | 预估行数 | MoonBit 特性运用 | 说明 |
|------|---------|----------------|------|
| `core/trait` — Serialize/Deserialize trait 定义 | ~500 | trait、泛型、关联类型 | 核心接口 |
| `core/derive` — derive 宏实现 | ~3000 | 元编程、代码生成 | 关键难点 |
| `core/serde` — JSON 后端适配器 | ~2000 | 模式匹配、ADT | 复用或重写 |
| `core/serde_msgpack` — MessagePack 后端 | ~3000 | 位操作、Bytes、模式匹配 | 完整二进制格式 |
| `core/testing` — 测试套件 | ~2000 | 泛型测试 | roundtrip 验证 |
| 示例和文档 | ~1000 | — | 使用示范 |

总预估：~10,000–12,000 行 MoonBit 代码

### 阶段划分

**Phase 1（基础）：**
- `Serializer` / `Deserializer` trait 定义
- `Serialize` / `Deserialize` 标记 trait
- 基础类型的内置实现（Bool, Int, String, Bytes, Option, Array, Map）
- JSON 后端（作为参考实现，用于验证 trait 设计正确性）
- roundtrip 测试

**Phase 2（derive）：**
- 研究 MoonBit derive 宏能力（当前 MoonBit 的 derive 是否可自定义，还是只能使用内置的）
- 实现 `derive(Serialize, Deserialize)` 宏
- struct 和 enum 的支持
- 属性配置（`#[serde(rename)]`、`#[serde(skip)]` 等）

**Phase 3（MessagePack 后端）：**
- 完整 MessagePack 编码/解码（基于现有社区库或重新实现）
- 接入 Serialize/Deserialize trait
- 性能优化

## 需要提前验证的关键技术问题

1. **MoonBit 是否支持自定义 derive 宏？** 
   — 检查 `derive` 机制是否可扩展。当前只看到内置的 `derive(Eq, Show, ToJson, FromJson)`，不确定用户能否添加 `derive(Serialize, Deserialize)`
   - 如果不能，替代方案：使用代码生成器（类似 protoc-gen-mbt 的方式）

2. **trait 泛型约束的灵活性**
   — `Serializer` 作为 trait 参数传递时，MoonBit 的类型系统是否支持

3. **特设多态（ad-hoc polymorphism）的限制**
   — MoonBit 的 `trait` 系统与 Rust 的相似度如何

### 关于 derive 宏的备选方案

如果 MoonBit 不支持自定义 derive 宏，替代路径：

- **方式 A**：使用代码生成器（类似 `protoc-gen-mbt`），通过外部工具生成序列化代码
- **方式 B**：使用 MoonBit 的 `moon.pkg` 虚拟包机制做依赖注入
- **方式 C**：手写 `impl Serialize for MyType`，依赖用户手动实现

推荐作法：先设计好 **trait 体系 + JSON 后端**，derive 作为追加优化。即使没有 derive 宏，通过手写 impl + 辅助宏也能减少样板代码。

---

## 比赛评审匹配度

| 评审维度 | 本项目如何满足 |
|----------|--------------|
| 解决实际应用问题 (25%) | MoonBit 生态最明显的缺口之一，每个需要网络通信、数据持久化的项目都依赖序列化 |
| 提供完整用户体验 (25%) | 提供 `derive` + 多格式后端 + 文档 + 示例，用户开箱即用 |
| 充分利用 MoonBit 语言特性 (25%) | 深度使用 trait、泛型、ADT、模式匹配、可能用到 derive 机制 |
| 结合领域特定知识和实际需求 (25%) | serde 是 Rust 生态最成功的库之一，将其设计理念迁移到 MoonBit 需要深入理解两种语言的类型系统差异 |

---

## 前期调研步骤（先做这些再写代码）

1. 搜索 mooncakes.io 确认 `serde` 关键词无同名包
2. 检查 MoonBit derive 机制的扩展性（查阅文档中关于 derive 的说明）
3. 检查 MoonBit `trait` 作为参数传递的限制
4. 拉取 `moonbit-community/msgpack` 看其实现质量，评估是否可复用/改进
5. 创建最小原型：定义 `Serialize` trait + JSON 编码器，验证设计可行性
