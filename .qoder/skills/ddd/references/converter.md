# 对象转换：Converter

对象跨层传递、需要变成另一层的对象时，交给专门的 Converter 完成。Controller、Listener、Case、Repository、Port 只调用 Converter，不在自己的方法里手写字段拷贝或 `builder()` 拼装另一层的对象。示例省略 package 和 import。

## 1. 位置与命名

| 边界 | 转换方向 | Converter | 位置 |
|------|----------|-----------|------|
| HTTP / MQ 入口 | api DTO、入站消息 ↔ 命令 / 领域结果 / ResponseDTO | `{Xxx}ApiConverter` | `trigger/converter/` |
| 跨领域编排 | A 领域的对象 → B 领域的命令 | `{Xxx}CaseConverter` | `cases/{domain}/{capability}/converter/` |
| 本地存储 | PO ↔ 实体 / 聚合 / 值对象 | `{Xxx}PoConverter` | `infrastructure/adapter/repository/converter/` |
| 远程调用 | 领域对象 ↔ 远程 DTO、MQ 消息体 | `{Xxx}PortConverter` | `infrastructure/adapter/port/converter/` |

- 按业务对象或外部系统拆分：一张表或一组紧密相关的表一个 PoConverter，一个外部系统一个 PortConverter，不建装下全部转换的万能 Converter。
- domain 不认识 DTO 和 PO，所以 domain 里没有 Converter。

## 2. 写法

| 约定 | 做法 |
|------|------|
| 形式 | `final` 类 + 私有构造 + `public static` 方法；不是 Spring Bean，不注入依赖 |
| 方法名 | `to{Target}`，列表用 `to{Target}List` |
| 只做映射 | 字段映射、类型转换、编码 ↔ 领域枚举（`fromCode`）、值对象 `of(...)` 重建；不查库、不调远程、不写业务规则 |
| 金额 | `MoneyVO.of(po.getTotalAmount(), po.getCurrency())`；反向取 `getAmount()`、`getCurrency()` |
| 空值 | 入参为 null 返回 null，列表为空返回空列表 |
| 不用反射拷贝 | 不用 `BeanUtils.copyProperties`：字段类型不同时静默丢值 |

工程已统一使用 MapStruct 等工具时沿用它，位置和职责规则不变。

不需要 Converter 的情况：只取一两个字段当参数传递；同一个对象原样往下传；领域内部的实体工厂、聚合工厂（它们带业务规则，留在 domain）。

## 3. 示例

### 3.1 PoConverter：PO ↔ 聚合

```java
public final class OrderPoConverter {

    private OrderPoConverter() {
    }

    public static OrderAggregate toAggregate(OrderPO orderPO, List<OrderItemPO> itemPOs) {
        if (orderPO == null) {
            return null;
        }
        OrderEntity order = OrderEntity.builder()
                .orderId(orderPO.getOrderId())
                .userId(orderPO.getUserId())
                .currency(orderPO.getCurrency())
                .totalAmount(MoneyVO.of(orderPO.getTotalAmount(), orderPO.getCurrency()))
                .status(OrderStatusEnumVO.fromCode(orderPO.getStatus()))   // 库表编码 → 领域枚举
                .build();
        List<OrderItemEntity> items = itemPOs == null ? new ArrayList<>() : itemPOs.stream()
                .map(po -> OrderItemEntity.builder()
                        .skuId(po.getSkuId())
                        .price(MoneyVO.of(po.getPrice(), orderPO.getCurrency()))
                        .quantity(po.getQuantity())
                        .build())
                .collect(Collectors.toCollection(ArrayList::new));   // 可变列表：聚合方法还要往里加明细
        return OrderAggregate.builder().order(order).items(items).build();
    }

    public static OrderPO toOrderPO(OrderAggregate aggregate) {
        if (aggregate == null) {
            return null;
        }
        OrderEntity order = aggregate.getOrder();
        return OrderPO.builder()
                .orderId(order.getOrderId())
                .userId(order.getUserId())
                .currency(order.getCurrency())
                .totalAmount(aggregate.totalAmount().getAmount())   // 金额取聚合的唯一计算结果
                .status(order.getStatus().getCode())
                .build();
    }

    // toItemPOList 写法相同
}
```

### 3.2 ApiConverter：DTO → 命令、领域结果 → DTO

```java
public final class OrderApiConverter {

    private OrderApiConverter() {
    }

    public static CreateOrderCommand toCommand(CreateOrderRequestDTO request) {
        if (request == null) {
            return null;
        }
        return CreateOrderCommand.builder()
                .userId(request.getUserId())
                .skuId(request.getSkuId())
                .quantity(request.getQuantity())
                .build();
    }

    public static CreateOrderResponseDTO toResponseDTO(OrderEntity order) {
        if (order == null) {
            return null;
        }
        return CreateOrderResponseDTO.builder()
                .orderId(order.getOrderId())
                .totalAmount(order.getTotalAmount().getAmount())   // 对外以分为单位
                .currency(order.getCurrency())
                .build();
    }
}
```

CaseConverter、PortConverter 写法相同：CaseConverter 只见两个领域的对象，PortConverter 只见远程 DTO、外部枚举和领域对象：外部状态先由外部枚举解析，再映射成领域枚举，写法见 [adapters.md](adapters.md) 3.1 节。

## 4. 反模式

```java
// ❌ Repository / Controller 里写私有转换方法，或现场 builder 拼装另一层的对象
private OrderAggregate toAggregate(OrderPO po) { ... }
CreateOrderCommand command = CreateOrderCommand.builder().userId(request.getUserId())...build();
// ✅ 经对应层的 Converter

// ❌ Converter 写业务规则：总额应由聚合计算
.totalAmount(po.getPrice() * po.getQuantity())
// ✅ Converter 只映射已有字段，计算放在实体或聚合里

// ❌ BeanUtils.copyProperties(po, entity)：status 在 PO 是 Integer、在实体是枚举，静默变成 null
// ✅ 手写映射，枚举用 fromCode
```
