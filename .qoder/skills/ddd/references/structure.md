# 工程结构与分层

六边形架构（端口与适配器）：domain 定义业务规则和对外依赖的接口；trigger、case 驱动 domain；infrastructure 实现 domain 的接口。示例根包为 `com.example`，按工程实际替换。

## 1. 依赖规则

```
                 app（依赖 trigger、infrastructure，负责装配）
                  │
                  ▼
   api ◀────── trigger ──────────────┐（无需编排时直接调领域服务）
                  │                  │
                  ▼                  ▼
                case ────────────▶ domain ◀────── infrastructure
```

| 模块 | 允许依赖 | 禁止依赖 |
|------|----------|----------|
| trigger | api、case、domain、types | infrastructure |
| case | domain、types | api、infrastructure |
| domain | types | 其他所有模块 |
| infrastructure | domain、types | api、case、trigger |
| api | types | 其他所有模块 |

domain 只使用 Spring 装配注解（`@Service`）、`@Resource`、Lombok 和通用工具库；不使用 ORM、Redis、HTTP、MQ 客户端，也不加 `@Transactional`。

各类对象能出现在哪些层：

| 对象 | 定义位置 | 可出现的层 |
|------|----------|------------|
| api 的 `{Xxx}RequestDTO` / `{Xxx}ResponseDTO` | api/dto | 只在 trigger，由 ApiConverter 转换 |
| `{Xxx}Entity` / `{Xxx}VO` / `{Xxx}Aggregate` / `{Action}Command` | domain/model | trigger、case、domain、infrastructure |
| `{Xxx}PO` 等本地存储对象 | infrastructure 的技术包，如 dao/po | 只在技术包、adapter/repository 及其 converter |
| 远程 DTO、MQ 消息体、外部枚举 | infrastructure 的技术包，如 gateway/dto | 只在技术包、adapter/port 及其 converter |

## 2. 限界上下文与防腐层

- 一个 `domain/{domain}` 包就是一个限界上下文，有自己的模型和 Repository。同一个业务概念在不同领域可以有不同的模型，不跨领域共用实体类。
- 跨领域协作只有三种方式：

| 需求 | 做法 |
|------|------|
| 同步地依次调用多个领域 | Case 依次调用各领域服务，跨域数据经 CaseConverter 转换 |
| 只读另一个领域的少量数据（同库） | 本领域 Repository 增加查询方法，返回本领域的对象 |
| 异步通知另一个领域或系统 | 写本地消息表并在提交后发送，对方的 Listener 消费后调用它自己的领域服务 |

- 防腐层就是 Port：`I{Xxx}Port` 的签名只出现领域对象；外部系统的字段、枚举、错误码在 adapter/port 内转换，不进入 domain。

## 3. 包结构

```
{project}-domain           com.example.domain.{domain}
├── adapter/
│   ├── repository/I{Domain}Repository.java
│   └── port/I{Xxx}Port.java（可选）
├── model/
│   ├── aggregate/{Xxx}Aggregate.java（可选）
│   ├── entity/{Xxx}Entity.java
│   ├── valobj/{Xxx}VO.java、{Xxx}EnumVO.java
│   └── command/{Action}Command.java
└── service/
    ├── I{Xxx}Service.java                    # 接口放 service 根目录，按业务能力划分
    └── {capability}/{Xxx}Service.java        # 实现

{project}-infrastructure   com.example.infrastructure
├── adapter/
│   ├── repository/
│   │   ├── {Domain}Repository.java           # 编排 DAO 等、本地事务、影响行数校验
│   │   └── converter/{Xxx}PoConverter.java
│   └── port/（可选）
│       ├── {Xxx}Port.java                    # 远程错误映射
│       └── converter/{Xxx}PortConverter.java
├── dao/
│   ├── I{Xxx}Dao.java                        # 通常一张表一个
│   └── po/{Xxx}PO.java
├── redis/（可选）                            # IRedisService 封装、连接配置
└── gateway/（可选）                          # {Xxx}GatewayService + dto/
```

api、case、trigger、app 的包结构见 SKILL.md。trigger 按入口类型分包（http / listener / job），不再按领域分。

## 4. infrastructure 分包规则

包按技术划分，数量不设上限：项目接入了什么依赖（数据库、Redis、ES、MongoDB、MQ、对象存储、外部 HTTP / RPC），就各建一个技术包。

| 包 | 角色 | 规则 |
|----|------|------|
| `adapter/repository`、`adapter/port` | 适配器，固定 | 唯一实现 domain 接口的地方；转换交给各自的 `converter/` |
| `dao/`、`redis/`、`es/` 等 | 本地存储类技术包 | 本服务拥有的数据；只被 Repository 调用 |
| `gateway/`、`mq/` 等 | 远程调用类技术包 | 数据归其他服务或外部系统；只被 Port 调用 |

- 技术包只处理连接、协议、序列化，不认识领域类型，不写业务规则，彼此不互相调用；对外不暴露底层客户端类型。
- 客户端 Bean 在 app 的 `config/` 声明。
- 禁止 `manager/`、`handler/` 这类同时调用多个技术包的编排包，编排放在领域服务；也不建与已有角色重复的包，如 `persistent/repository/`。

## 5. 命名

| 类型 | 规则 | 示例 |
|------|------|------|
| 实体 / 值对象 / 领域枚举 | `{Xxx}Entity` / `{Xxx}VO` / `{Xxx}EnumVO` | `OrderEntity`、`MoneyVO`、`OrderStatusEnumVO` |
| 聚合 / 命令 | `{Xxx}Aggregate` / `{Action}Command` | `OrderAggregate`、`CreateOrderCommand` |
| Repository / Port | `I{Domain}Repository` / `{Domain}Repository`；`I{Xxx}Port` / `{Xxx}Port`（按外部能力命名） | `IOrderRepository`、`IProductPort` |
| 领域服务 / Case | `I{Xxx}Service` / `{Xxx}Service`；`I{Xxx}Case` / `{Xxx}Case` | `IOrderService`、`IPaySuccessCase` |
| DAO / PO | `I{Xxx}Dao` / `{Xxx}PO` | `IOrderDao`、`OrderPO` |
| 远程客户端 / 远程 DTO | `{Xxx}GatewayService` / 以外部系统命名的 `{External}RequestDTO` | `ProductGatewayService`、`ProductResponseDTO` |
| 外部枚举 | `{External}{Xxx}Enum`，以外部系统命名，不带 `VO` 后缀，不与领域枚举同名 | `ProductCenterStatusEnum` |
| Converter | `{Xxx}ApiConverter` / `{Xxx}CaseConverter` / `{Xxx}PoConverter` / `{Xxx}PortConverter` | `OrderPoConverter` |
| 入口 | `{Xxx}Controller` / `{Xxx}Listener` / `{Xxx}Job` | `OrderController` |

| 方法场景 | 规则 | 示例 |
|----------|------|------|
| Repository 查询 | `query{Xxx}By{Yyy}`；读取整个聚合用 `findBy{Yyy}` | `queryOrderByOrderId`、`findByOrderId` |
| Repository 写入 | 整个聚合用 `save` / `update`；其他按业务动作命名 | `save(OrderAggregate)`、`paySuccess` |
| 领域服务、Case、Port | 业务动作，不用 `execute`、`process` | `createOrder`、`queryProduct` |
| 判断 / 校验 | `is` / `can` / `has`；不通过即抛异常用 `check` | `canCancel`、`checkPayable` |
| 转换 | `to{Target}` | `toEntity`、`toPO`、`toCommand` |

## 6. 常见分层错误

```java
// ❌ 领域服务入参是 api 的 DTO，domain 因此依赖了 api
OrderEntity createOrder(CreateOrderRequestDTO request);
// ✅ trigger 把 DTO 转成命令
OrderEntity createOrder(CreateOrderCommand command);

// ❌ 订单域的服务注入积分域的 Repository
@Resource
private ICreditRepository creditRepository;
// ✅ 由 Case 依次调用订单域、积分域的领域服务

// ❌ Repository 调用 gateway：远程调用进了本地事务
public class OrderRepository implements IOrderRepository {
    @Resource
    private ProductGatewayService productGatewayService;
}
// ✅ 领域服务先经 IProductPort 取得商品，再把需要的数据放进聚合交给 Repository
```
