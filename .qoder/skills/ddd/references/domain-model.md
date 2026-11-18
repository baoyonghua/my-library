# 领域模型：实体、值对象与命令

实体看标识（"是不是同一个"），值对象看属性值（属性全相同即可互换），命令只是一次调用的入参。示例统一用订单领域，省略 package 和 import。

## 1. 放置

| 包 | 放什么 |
|----|--------|
| `model/entity/` | 有唯一标识、有生命周期的实体 |
| `model/valobj/` | 值对象 `{Xxx}VO` 和领域枚举 `{Xxx}EnumVO`；不建子包，不嵌套在实体或聚合类里 |
| `model/command/` | 领域服务的入参 `{Action}Command` |
| types 模块 | 只放跨层共用的枚举，如 `ErrorCode` |

## 2. 实体

实体跨请求存活、被持久化，必须有行为：状态变更、业务判断、派生计算。规则收在实体里，领域服务只负责编排。

- 注解 `@Data @Builder @AllArgsConstructor @NoArgsConstructor`。setter 只用于组装和转换（如 PoConverter 把 PO 转成实体），状态变更必须走业务方法。
- 需要按身份比较（集合去重、作 Map key）时，加 `@EqualsAndHashCode(onlyExplicitlyIncluded = true)`，标识字段加 `@EqualsAndHashCode.Include`。
- 只放业务数据和业务标识；Redis key、表名、PO、JSON 串属于基础设施细节，不进入实体。

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class OrderEntity {

    @EqualsAndHashCode.Include
    private String orderId;
    private String userId;
    private MoneyVO totalAmount;
    private OrderStatusEnumVO status;

    /** 新建订单：初始状态由实体决定，不交给调用方填写 */
    public static OrderEntity create(String orderId, String userId) {
        return OrderEntity.builder()
                .orderId(orderId)
                .userId(userId)
                .totalAmount(MoneyVO.zero(currency))
                .status(OrderStatusEnumVO.CREATE)
                .build();
    }

    public void pay() {
        if (!status.canPay()) {
            throw new BusinessException(ErrorCode.ILLEGAL_STATE.getCode(), "订单状态不允许支付: " + orderId);
        }
        this.status = OrderStatusEnumVO.PAID;
    }

    public void cancel() {
        if (!status.canCancel()) {
            throw new BusinessException(ErrorCode.ILLEGAL_STATE.getCode(), "订单状态不允许取消: " + orderId);
        }
        this.status = OrderStatusEnumVO.CANCELLED;
    }
}
```

## 3. 值对象

| 约定 | 做法 |
|------|------|
| 不可变 | 不用 `@Data`，没有 setter；任何"修改"都返回新实例 |
| 值相等 | `@EqualsAndHashCode` 比较全部字段 |
| 创建 | 没有校验规则用 `@Builder`；有校验规则只提供静态工厂 `of(...)`，不暴露 builder |
| 校验 | 在工厂里完成，失败抛 `BusinessException`；创建出来的实例一定合法 |
| 注解 | `@Getter @EqualsAndHashCode @AllArgsConstructor(access = PRIVATE) @NoArgsConstructor(access = PRIVATE)` |

### 金额：MoneyVO

```java
@Getter
@EqualsAndHashCode
@AllArgsConstructor(access = AccessLevel.PRIVATE)
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class MoneyVO {

    /** 金额，单位是该币种的最小单位（人民币为分），1500 表示 15.00 元 */
    private long amount;
    /** 币种，ISO 4217 代码，如 CNY */
    private String currency;

    /** 参数用 Long，能直接接收 PO / DTO 的可空字段，空值和负数在这里统一拦截 */
    public static MoneyVO of(Long amount, String currency) {
        if (amount == null || amount < 0) {
            throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "金额不能为空或负数");
        }
        if (currency == null || currency.isBlank()) {
            throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "币种不能为空");
        }
        return new MoneyVO(amount, currency);
    }

    public static MoneyVO zero(String currency) {
        return of(0L, currency);
    }

    /** 溢出时 addExact 抛异常，不会静默变成负数 */
    public MoneyVO add(MoneyVO other) {
        assertSameCurrency(other);
        return of(Math.addExact(amount, other.amount), currency);
    }

    public MoneyVO multiply(int quantity) {
        return of(Math.multiplyExact(amount, (long) quantity), currency);
    }

    public boolean isPositive() {
        return amount > 0;
    }

    private void assertSameCurrency(MoneyVO other) {
        if (other == null || !currency.equals(other.currency)) {
            throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "币种不一致，不能运算");
        }
    }
}
```

- 内部用 `long` 存最小货币单位：整数运算没有精度误差；运算前校验币种。
- 数据库存 `BIGINT` 金额列 + 币种列；PO 和 DTO 用 `Long` 金额（分）+ `String currency`，由 Converter 与 `MoneyVO` 互转。
- 折扣、费率等比例计算放在对应的值对象里，显式指定取整方式，结果仍返回 `MoneyVO`。

### 领域枚举

- `@Getter @AllArgsConstructor`，字段为 `code` + `desc`，声明 `final`。
- 提供 `fromCode(code)`：遍历 `values()` 用 `equals` 比较；未知 code 或 null 抛 `BusinessException`，不返回默认值。
- 可以有只依赖自身的判断方法。

```java
@Getter
@AllArgsConstructor
public enum OrderStatusEnumVO {

    CREATE(0, "新建"),
    PAID(1, "已支付"),
    CANCELLED(2, "已取消"),
    ;

    private final Integer code;
    private final String desc;

    public static OrderStatusEnumVO fromCode(Integer code) {
        for (OrderStatusEnumVO value : values()) {
            if (value.code.equals(code)) {
                return value;
            }
        }
        throw new BusinessException(ErrorCode.ILLEGAL_PARAMETER.getCode(), "未知的订单状态: " + code);
    }

    public boolean canPay() {
        return this == CREATE;
    }

    public boolean canCancel() {
        return this == CREATE;
    }
}
```

## 4. 命令

命令表达"调用方要求领域做一件事"的输入：没有标识、不持久化、不对外返回，所以既不是实体也不是值对象。

| 约定 | 做法 |
|------|------|
| 位置 | `domain/{domain}/model/command/`。领域服务签名要用它，而 case 只依赖 domain，所以只能定义在 domain |
| 命名 | `{Action}Command`，不加 `Entity` 后缀 |
| 注解 | `@Data @Builder @AllArgsConstructor @NoArgsConstructor` |
| 字段 | 简单值或本领域的实体 / 值对象；不嵌套 api DTO、PO |
| 内容 | 只放调用方真正掌握的输入；库里已有的数据（如商品价格）由领域服务自己加载，否则调用方可以伪造 |
| 创建 | 由 trigger / case 经 Converter 从 DTO 或其他领域的结果转换而来 |

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class CreateOrderCommand {

    private String userId;
    private String skuId;
    private Integer quantity;
    /** 不放单价：价格由领域服务经 IProductPort 查询，不信任调用方传入 */
}
```

## 5. 反模式

```java
// ❌ 可变值对象：@Data 生成 setter，add 原地修改，所有共享该实例的地方都被改掉
@Data
public class MoneyVO {
    private long amount;
    public void add(MoneyVO other) { this.amount += other.amount; }
}
// ✅ 无 setter，运算返回新实例

// ❌ 有校验规则的值对象加 @Builder：调用方可跳过 of() 造出负金额
MoneyVO money = MoneyVO.builder().amount(-1L).build();
// ✅ 只保留静态工厂
MoneyVO money = MoneyVO.of(1000L, "CNY");

// ❌ 枚举查找静默兜底：未知 code 被当成 CREATE，脏数据流入业务
default: return CREATE;
// ✅ fromCode 对未知或 null 抛 BusinessException

// ❌ 服务里直接判断实体字段：规则散落，换个入口就要再写一遍
if (order.getStatus() == OrderStatusEnumVO.CREATE) { order.setStatus(OrderStatusEnumVO.PAID); }
// ✅ 规则收在实体里
order.pay();
```
