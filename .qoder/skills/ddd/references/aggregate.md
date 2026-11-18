# 聚合

聚合是一组必须保持一致的对象的边界，也是一次本地事务的边界。调用链固定为：领域服务组装或加载聚合 → 调用 Repository → Repository 实现在一个本地事务内完成全部写入。示例省略 package 和 import。

## 1. 术语

- **聚合根**：边界内带唯一标识、对外唯一的入口实体。`{Xxx}Aggregate` 是容器，聚合根是它持有的主实体（如 `OrderEntity`）。
- **非根实体**（如 `OrderItemEntity`）不能被外部直接获取、修改或单独保存，只能通过聚合根的方法修改。

## 2. 判断是否需要聚合

| 情况 | 结论 |
|------|------|
| 只写单个实体对应的数据 | 不需要，Repository 直接接收实体 |
| 一次操作要原子地写多张表或多条记录 | 需要，用**组装型** |
| 多个对象间有必须始终成立的规则（如总额 = 明细之和） | 需要，用**行为型** |
| 涉及多个聚合，但不需要强一致 | 不合并成大聚合；分别保存，用本地消息表 / MQ 做最终一致 |
| 纯查询 | 不需要 |

划定边界的标准是"是否必须在同一个本地事务里一起成功或失败"，而不是"这次操作是否用到了它"。其他聚合的数据只放 ID 或必要字段；外部调用、Redis 不进入聚合。

## 3. 组装型与行为型

| | 组装型 | 行为型 |
|--|--|--|
| 用途 | 一次原子写入的入参载体 | 成员之间存在不变量 |
| 聚合根 | 不强制（项目约定，评审时不判错） | 必须有 |
| 业务方法 | 可以没有 | 所有修改都通过业务方法，在方法内校验不变量 |
| 注解 | `@Data @Builder @AllArgsConstructor @NoArgsConstructor`，只在一次调用内存活 | 不用 `@Data`，不提供公开 setter；集合只读暴露 |
| Repository | 按业务动作命名，如 `paySuccess(aggregate)` | `findByXxx` / `save` / `update`，以整个聚合为单位读写 |

## 4. 示例：行为型订单聚合

不变量：至少有一条明细；总额始终等于各明细"单价 × 数量"之和；明细单价与订单币种一致；只有新建状态的订单能修改明细。

```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class OrderAggregate {

    private OrderEntity order;   // 聚合根
    @Getter(AccessLevel.NONE)
    @Builder.Default
    private List<OrderItemEntity> items = new ArrayList<>();

    public List<OrderItemEntity> getItems() {
        return Collections.unmodifiableList(items);
    }

    public void addItem(OrderItemEntity item) {
        if (item == null || item.getQuantity() == null || item.getQuantity() <= 0) {
            throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "订单明细不合法");
        }
        if (!order.getCurrency().equals(item.getPrice().getCurrency())) {
            throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "明细币种与订单不一致");
        }
        if (order.getStatus() != OrderStatusEnumVO.CREATE) {
            throw new BusinessException(ErrorCode.ILLEGAL_STATE.getCode(), "当前订单状态不允许修改明细");
        }
        items.add(item);
        order.setTotalAmount(totalAmount());   // 聚合根维护不变量，允许在这里调用 setter
    }

    /** 唯一的金额计算，其他地方都调用它 */
    public MoneyVO totalAmount() {
        return items.stream()
                .map(i -> i.getPrice().multiply(i.getQuantity()))
                .reduce(MoneyVO.zero(order.getCurrency()), MoneyVO::add);
    }

    public void validate() {
        if (items.isEmpty() || !totalAmount().isPositive()) {
            throw new BusinessException(ErrorCode.ILLEGAL_STATE.getCode(), "订单没有明细或金额不大于 0");
        }
    }
}
```

`@Builder` 和无参构造只供 PoConverter 从 PO 重建聚合时使用，业务代码只通过聚合方法修改。领域服务的写法：`findByOrderId` 读出整个聚合 → 调用聚合方法 → `update` 整体写回，见 [domain-service.md](domain-service.md)。

## 5. 示例：组装型

订单支付成功时，要在一个事务里同时更新订单状态和写入支付记录，两者没有不变量，只是"必须一起成功"，用组装型：

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class OrderPaySuccessAggregate {

    private String orderId;
    private String tradeNo;                 // 支付流水号
    private OrderPayRecordEntity payRecord; // 与状态更新同一事务写入的支付记录
}
```

对应的 Repository 方法 `boolean paySuccess(OrderPaySuccessAggregate aggregate)` 在一个事务里条件更新订单状态（`WHERE status = CREATE`）并插入支付记录；已支付过返回 `false`，作为幂等空操作。

## 6. 反模式

```java
// ❌ 绕过聚合根修改内部对象：总额、状态校验都不会执行
aggregate.getItems().get(0).setQuantity(5);
orderItemRepository.updateQuantity(itemId, 5);
// ✅ 只通过聚合方法修改，再整体保存

// ❌ 操作用到了就合成大聚合：用户、商品有独立生命周期，不需要与订单强一致
public class OrderAggregate {
    OrderEntity order;
    UserEntity user;
    ProductEntity product;
}
// ✅ 只保留必须同事务写入的对象，其他聚合按 ID 引用（order 里放 userId，明细里放 skuId）

// ❌ 行为型聚合用 @Data：公开 setter 让任何人都能 setItems(...)，业务方法形同虚设
// ✅ @Getter + 私有全参构造，集合只读暴露

// ❌ 让调用方构建含领域计算值的聚合（如总额、校验结果），调用方可以伪造
// ✅ 调用方只传命令，聚合由领域服务在校验后组装
```
