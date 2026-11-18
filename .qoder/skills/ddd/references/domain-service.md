# 领域服务

领域服务编排本领域的一次业务操作：读数据（Repository）、调外部（Port）、驱动实体和聚合的行为、写回数据。
业务规则本身放在实体、值对象、聚合上，领域服务不重复实现。示例省略 package 和 import。

## 1. 逻辑放在哪里

先把逻辑放到最近的对象上，放不下再放服务。

| 逻辑特征                   | 放在哪里                      | 示例                                |
|----------------------------|-------------------------------|-------------------------------------|
| 只用到单个对象自身的数据   | 实体 / 值对象方法             | `order.pay()`、`status.canCancel()` |
| 聚合内多个对象之间的不变量 | 聚合方法                      | `aggregate.addItem(item)`           |
| 需要读写数据或调用外部     | 领域服务                      | 创建订单、取消订单                  |
| 跨领域编排                 | Case，见 [entry.md](entry.md) | 支付成功后发放积分                  |

信号：服务里出现只读取同一个实体字段的判断，说明规则应该移到实体上。领域服务是无状态单例，不持有业务数据。

## 2. 位置与命名

- 接口 `service/I{Capability}Service`，按业务能力划分；实现 `service/{capability}/{Capability}Service`，标注 `@Service`。
- 方法名用业务动词：`createOrder`、`cancelOrder`；`save` / `update` / `find` 是 Repository 的词汇。
- 返回领域对象或基础类型，不返回 PO、DTO；失败抛 `BusinessException`。

## 3. 入参

| 入参情况                           | 做法                                               |
|------------------------------------|----------------------------------------------------|
| 1～2 个含义独立的标识或简单值      | 直接传参，如 `cancelOrder(String orderId)`         |
| 多个输入共同描述一次操作，或会扩展 | 命令，如 `createOrder(CreateOrderCommand command)` |

含领域计算值（总额、校验结果）的聚合必须由领域服务自己组装，不能让调用方构建。

## 4. 依赖

| 可以依赖                                                                           | 不可以依赖                                 |
|------------------------------------------------------------------------------------|--------------------------------------------|
| 本领域的 Repository、Port 接口                                                     | 其他领域的 Repository、Port、领域服务      |
| 本领域的实体、值对象、聚合、命令                                                   | infrastructure 的任何类（DAO、客户端、PO） |
| types 的 `BusinessException`、`ErrorCode`；通用的 `MoneyVO`；本地消息等与业务无关的通用组件接口 | api 的 DTO                                 |

跨领域需要对方的计算结果时，由 Case 先调用对方领域，把结果放进本领域的命令；数据归其他服务所有时，经本领域的 Port 获取。

## 5. 示例

```java
@Service
public class OrderService implements IOrderService {

    @Resource
    private IOrderRepository repository;
    @Resource
    private IProductPort productPort;

    @Override
    public OrderEntity createOrder(CreateOrderCommand command) {
        // 1. 读外部数据：在事务之外调用 Port，价格以商品服务为准
        ProductVO product = productPort.queryProduct(command.getSkuId());
        if (product == null || !product.isOnSale()) {
            throw new BusinessException(ErrorCode.ILLEGAL_STATE.getCode(), "商品不可购买: " + command.getSkuId());
        }
        
        // 2. 组装聚合
        OrderEntity order = OrderEntity.create(command.getUserId(), product.getPrice());
        OrderItemEntity item = OrderItemEntity.create(product.getSkuId(), product.getPrice(), command.getQuantity());
        OrderAggregate aggregate = OrderAggregate.builder()
                .order(order)
                .items(List.of(item))
                .build();
        aggregate.validate();
        
        // 3. 写入：@Transactional 在 Repository 实现上
        repository.save(aggregate);
        return aggregate.getOrder();
    }

    @Override
    public void cancelOrder(String orderId) {
        OrderAggregate aggregate = repository.findByOrderId(orderId);
        if (aggregate == null) {
            throw new BusinessException(ErrorCode.ORDER_NOT_EXIST.getCode(), "订单不存在: " + orderId);
        }
        aggregate.getOrder().cancel();   // 状态规则在实体里
        repository.update(aggregate);
    }
}
```

## 6. 复杂度控制

- 方法保持"读 → 调用模型行为 → 写"的步骤清单；步骤多时拆成有业务含义的私有方法。
- 同一操作按类型出现多段大块分支、且还会继续增加时，再考虑把分支拆成独立的类；只有一两种情况时直接写 `if`
  ，不为"将来可能变"提前抽象。

## 7. 反模式

```java
// ❌ 一个 IOrderService 装下创建、支付、退款、结算全部操作
// ✅ 按业务能力拆分接口：IOrderService、IOrderPayService、IOrderRefundService

// ❌ 规则写在服务里：状态判断、金额计算平铺在服务方法中
if(order.getStatus() ==OrderStatusEnumVO.CREATE){...}
long total = items.stream().mapToLong(i -> i.getPrice() * i.getQuantity()).sum();
// ✅ order.cancel()、aggregate.totalAmount()

// ❌ 在领域服务上加 @Transactional，把 Port 调用一起包进事务
// ✅ 事务只在 Repository 实现上，见 consistency.md
```
