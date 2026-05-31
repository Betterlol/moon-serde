# moon-serde

通用序列化框架 — MoonBit 生态的 serde 等价物。

## 目标

为 MoonBit 提供一套**格式无关的序列化抽象层**，让用户通过 `derive(Serialize, Deserialize)` 一行注解即可获得多格式的序列化能力。

```moonbit
struct User {
  name : String
  age : Int
  email : String?
} derive(Serialize, Deserialize)

// JSON
let json_bytes = @serde::to_json(user)
let user_back : User = @serde::from_json(json_bytes)

// MessagePack
let msgpack_bytes = @serde::to_msgpack(user)
let user_back2 : User = @serde::from_msgpack(msgpack_bytes)
```

## 现状

MoonBit 当前有三个独立的序列化方案（JSON / protobuf / MessagePack），各有各的 trait 和 enum，彼此不互通。缺少一个统一的 `Serialize`/`Deserialize` trait 体系来解耦"数据结构"和"序列化格式"。

**moon-serde 要填的正是这个空白。**

## 架构

```
用户类型 → derive(Serialize, Deserialize) → Serialize/Deserialize trait
                                                   │
                          ┌────────────────────────┼──────────────┐
                          ▼                        ▼              ▼
                   JSON 后端               MsgPack 后端      BSON 后端
```

## 开发状态

项目处于 **设计调研阶段**，详见 [instr.md](instr.md)。

## 2026 MoonBit 国产基础软件开源大赛

本项目参与 [CCF × MoonBit 国产基础软件开源大赛](https://www.gitlink.org.cn/competitions/track1_2026MoonBit)。
