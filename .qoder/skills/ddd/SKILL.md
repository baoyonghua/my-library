---
name: ddd
description: Java DDD 六边形架构的设计与编码规范（分层、包结构、实体 / 值对象 / 聚合、领域服务、Repository / Port、Converter、事务与一致性）。在 DDD 分层工程中新增或修改功能、设计领域模型、判断代码应放在哪一层或哪个包、评审分层约束时使用。工程含 -domain / -infrastructure / -trigger 模块，或 adapter/repository、model/entity 等包，或 I{Xxx}Repository、{Xxx}Entity 等命名时，即视为 DDD 工程。
---

# DDD 六边形架构

按固定的分层、包结构和约束，在 DDD 多模块工程中设计并实现功能。

## 使用前确认

满足任意一类特征即视为 DDD 工程：

- **模块**：存在 `{project}-domain`、`{project}-infrastructure`、`{project}-trigger`，常伴随 `-api`、`-case`、`-types`、`-app`。
- **包**：存在 `domain/{领域}/adapter/repository`、`model/entity`、`model/valobj` 等。
- **命名**：存在 `I{Xxx}Repository`、`I{Xxx}Port`、`{Xxx}Entity`、`{Xxx}VO` 等。

| 结果                                                | 做法                                                                                                             |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 符合                                                | 先读已有的 domain、infrastructure、trigger，沿用其命名和写法；工程约定与本规范冲突时以工程为准，并向用户说明差异 |
| 不符合（如 controller / service / mapper 三层工程） | 不套用本规范，先询问用户是否按 DDD 组织                                                                          |
| 空工程且用户明确要求 DDD                            | 按本规范创建模块和包，并告知用户创建了哪些                                                                       |

## 分层与依赖

```
trigger ──→ api
   │
   ├──→ case（可选）──→ domain ←── infrastructure
   │                      ▲
   └──────────────────────┘
types：所有模块都可依赖      app：启动入口与装配
```

| 层             | 职责                                                                                 | 禁止                                                |
|----------------|--------------------------------------------------------------------------------------|-----------------------------------------------------|
| api            | 对外接口、`{Xxx}RequestDTO` / `{Xxx}ResponseDTO`、`Response<T>`                      | 业务逻辑                                            |
| trigger        | 接收 HTTP / MQ / Job；经 ApiConverter 做 DTO ↔ 命令 / 结果转换；调用 Case 或领域服务 | 业务逻辑；注入 Repository / Port / DAO；开事务      |
| case           | 跨领域编排多个领域服务                                                               | 调用 Repository / Port；开事务；写业务规则          |
| domain         | 实体、值对象、聚合、领域服务；定义 Repository / Port 接口                            | 依赖 infrastructure、api；注入其他领域的 Repository |
| infrastructure | 实现 Repository / Port；访问数据库、缓存、外部服务                                   | 业务判断                                            |
| types          | `BusinessException`、`ErrorCode`、通用常量                                           | 业务逻辑                                            |

## 包结构速览

示例根包 `com.example`，按工程实际替换。标"（可选）"的只在用到时创建，不预先建空包。

```
{project}-types            com.example.types
├── enums/ErrorCode.java
└── exception/BusinessException.java

{project}-api（可选）      com.example.api
├── I{Capability}Service.java                 # 对外接口，不与领域服务接口同名
├── dto/{Xxx}RequestDTO.java、{Xxx}ResponseDTO.java
└── response/Response.java

{project}-domain           com.example.domain.{domain}
├── adapter/
│   ├── repository/I{Domain}Repository.java   # 本服务数据的访问接口
│   └── port/I{Xxx}Port.java（可选）          # 外部系统的调用接口
├── model/
│   ├── aggregate/{Xxx}Aggregate.java（可选）
│   ├── entity/{Xxx}Entity.java               # 有唯一标识的实体
│   ├── valobj/{Xxx}VO.java、{Xxx}EnumVO.java # 值对象、领域枚举
│   └── command/{Action}Command.java          # 领域服务入参
└── service/
    ├── I{Xxx}Service.java
    └── {capability}/{Xxx}Service.java

{project}-case（可选）     com.example.cases.{domain}.{capability}    # case 是关键字，包名用 cases
├── I{Xxx}Case.java、{Xxx}Case.java
└── converter/{Xxx}CaseConverter.java

{project}-infrastructure   com.example.infrastructure
├── adapter/
│   ├── repository/{Domain}Repository.java、converter/{Xxx}PoConverter.java
│   └── port/{Xxx}Port.java、converter/{Xxx}PortConverter.java
├── dao/、dao/po/                             # 以下为技术包，一种技术一个包，按需新建
├── redis/、es/ …                             # 本地存储：只被 Repository 调用
└── gateway/、mq/ …                           # 远程调用：只被 Port 调用

{project}-trigger          com.example.trigger
├── http/{Xxx}Controller.java
├── listener/{Xxx}Listener.java（可选）
├── job/{Xxx}Job.java（可选）
└── converter/{Xxx}ApiConverter.java

{project}-app              com.example
├── Application.java
└── config/                                   # Spring 配置、全局异常处理
```

完整的模块、包与命名规范见 [references/structure.md](references/structure.md)。

## 强制约束

1. **命名**：接口以 `I` 开头；实现类（Repository、Port、Service、Case）不带 `Impl` 后缀。
2. **domain 只依赖接口**：领域服务只注入本领域的 Repository / Port 接口，同领域其他服务只通过 `I{Xxx}Service`
   接口调用且不循环依赖；跨领域交给 Case 编排。例外：本地消息等与业务无关的通用组件接口可以注入。
3. **infrastructure 按技术分包**：`adapter/repository`、`adapter/port` 是唯一实现 domain 接口的地方；技术包（dao、redis、es、mq、gateway
   等）数量不限，只封装客户端，彼此不互相调用。本地存储类只被 Repository 调用，远程调用类只被 Port 调用。不建 `manager/`
   这类编排包。
4. **事务**：`@Transactional(rollbackFor = Exception.class)` 只加在 Repository 实现的方法上，一个方法对应一次聚合写入；领域服务、Case、Trigger
   不加。
5. **事务内不做远程调用**：RPC / HTTP / MQ 放在事务之外。需要写库后可靠地发送 MQ 时，用全服务共用的本地消息表：消息与业务数据同一事务写入，提交后发送，失败由定时任务重试。工程已有本地消息组件或其他可靠投递方案时沿用工程的。
6. **非事务存储要补偿**：Redis、ES 等不参与数据库事务，提交后写入或失败时补偿，用过期时间或对账兜底。
7. **层间转换走 Converter**：DTO ↔ 命令（ApiConverter）、跨域对象（CaseConverter）、PO ↔ 领域对象（PoConverter）、远程 DTO ↔
   领域对象（PortConverter）。Converter 是 `final` 类 + 静态方法，只做映射，不注入依赖、不写业务规则。
8. **异常**：领域内抛 `BusinessException(ErrorCode.Xxx.getCode(), message)`；有全局异常处理时 Trigger 不捕获；MQ
   监听器不吞异常。文档中 `SUCCESS`、`UN_ERROR`、`ILLEGAL_PARAMETER` 以外的错误码都是示例，按工程实际的 `ErrorCode` 补充。

## 新功能开发流程

1. **确定归属领域**：现有领域能支持就在其内扩展；全新业务才新建领域。
2. **确定入口与是否需要 Case**：单领域业务由 Trigger 直接调用领域服务；需要编排多个领域时才建 Case。已有 Trigger
   变复杂时，先询问用户是否重构为 Case。详见 [references/entry.md](references/entry.md)。
3. **设计领域模型**：

   | 需要表达的东西 | 用 |
      |----------------|----|
   | 有唯一标识、有生命周期的业务对象 | 实体 `{Xxx}Entity` |
   | 无标识、不可变、按值相等的概念；状态与类型 | 值对象 `{Xxx}VO`、`{Xxx}EnumVO` |
   | 领域服务的入参（多个相关输入） | 命令 `{Action}Command` |
   | 一次原子写入涉及多个对象，或对象间有必须始终成立的规则 | 聚合 `{Xxx}Aggregate` |

   详见 [references/domain-model.md](references/domain-model.md)、[references/aggregate.md](references/aggregate.md)。
4. **实现领域服务**：规则优先放在实体和聚合上，领域服务只编排"读 → 调用模型行为 →
   写"。详见 [references/domain-service.md](references/domain-service.md)。
5. **定义并实现适配器**：本服务的数据走 Repository，跨进程调用走
   Port；接口只用领域对象。详见 [references/adapters.md](references/adapters.md)、[references/converter.md](references/converter.md)。
6. **处理一致性**：涉及多次写入、远程调用、缓存、通知时，按 [references/consistency.md](references/consistency.md) 处理。

## 自检清单

- [ ] 代码在正确的模块和包里
- [ ] Trigger 没有业务逻辑，没有注入 Repository / Port / DAO
- [ ] Case 只调用领域服务，并完成各个领域服务之间的编排，没有加事务，没有写业务规则
- [ ] 领域服务只依赖本领域的 Repository / Port 接口；业务规则在实体 / 聚合上
- [ ] `@Transactional` 只在 Repository 实现上，事务内没有 RPC / HTTP / MQ
- [ ] Repository / Port 接口只出现领域对象；infrastructure 没有业务判断
- [ ] 外部服务返回的状态用外部枚举表示，在适配器里映射成领域枚举，映射不写 `default`
- [ ] 层间转换都经 Converter；
- [ ] 写库后可靠发送 MQ 走本地消息表，消费方按消息 ID 幂等并能处理乱序
- [ ] 异常统一使用 `BusinessException`，MQ 监听器没有吞异常

## 参考资料

| 文件                                                         | 何时阅读                                        |
|--------------------------------------------------------------|-------------------------------------------------|
| [references/structure.md](references/structure.md)           | 确认模块依赖、包路径、命名、infrastructure 分包 |
| [references/domain-model.md](references/domain-model.md)     | 设计实体、值对象、金额、枚举、命令              |
| [references/aggregate.md](references/aggregate.md)           | 判断是否需要聚合，设计聚合                      |
| [references/domain-service.md](references/domain-service.md) | 编写领域服务                                    |
| [references/adapters.md](references/adapters.md)             | 定义或实现 Repository、Port                     |
| [references/converter.md](references/converter.md)           | 编写层间对象转换                                |
| [references/entry.md](references/entry.md)                   | 编写 Case、Controller、Listener、Job            |
| [references/consistency.md](references/consistency.md)       | 事务边界、远程调用、本地消息表、缓存、幂等      |
