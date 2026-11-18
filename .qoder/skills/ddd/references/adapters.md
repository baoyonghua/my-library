# 适配器：Repository 与 Port

domain 在 `domain/{domain}/adapter/` 下定义接口，infrastructure 在 `adapter/`
下实现。包规则见 [structure.md](structure.md) 第 4 节，转换写法见 [converter.md](converter.md)
，事务与一致性见 [consistency.md](consistency.md)。示例省略 package 和 import。

## 1. Repository 还是 Port

**本服务的数据走 Repository，跨进程的一律走 Port。**

| 要访问的对象                                                        | 选择                       |
|---------------------------------------------------------------------|----------------------------|
| 本服务的数据库表、Redis、ES 索引等                                  | Repository                 |
| 其他服务的数据（即使物理上能连到它的库）、配置中心、外部 HTTP / RPC | Port                       |
| 发送 MQ、HTTP 回调                                                  | Port，只能在事务提交后执行 |

## 2. Repository

实现类 `{Domain}Repository` 标注 `@Repository`。负责本服务数据的读写、一次聚合写入的本地事务、条件更新的影响行数校验；PO ↔
领域对象的转换交给 PoConverter。不负责业务规则和远程调用。

接口设计：

- 参数和返回值只用领域类型，不出现 PO、DTO、`Map`；只放领域服务实际用到的方法。
- 行为型聚合以整个聚合读写：`findByOrderId` / `save` / `update`；其他写方法按业务动作命名，一个方法对应一次原子写入。
- 查询方法表达业务含义（`queryOrderByOrderId`），不提供 `find(String query)`、`findByCondition(Map)` 这类通用查询。
- 影响行数在实现内校验，0 行抛 `BusinessException`；约定为幂等空操作的方法返回 `false`，并在接口注释写明。

```java
public interface IOrderRepository {

    String nextOrderId();

    /** 读取整个订单聚合（主表 + 明细）；不存在返回 null */
    OrderAggregate findByOrderId(String orderId);

    void save(OrderAggregate aggregate);

    void update(OrderAggregate aggregate);

    OrderEntity queryOrderByOrderId(String orderId);

    /** 订单改为已支付并写入支付记录，同一事务；已支付过返回 false（幂等空操作） */
    boolean paySuccess(OrderPaySuccessAggregate aggregate);
}
```

```java
@Repository
public class OrderRepository implements IOrderRepository {

    @Resource
    private IOrderDao orderDao;
    @Resource
    private IOrderItemDao orderItemDao;

    @Override
    public OrderAggregate findByOrderId(String orderId) {
        OrderPO orderPO = orderDao.selectByOrderId(orderId);
        if (orderPO == null) {
            return null;
        }
        // 明细一次查出，不在循环里逐条查库
        return OrderPoConverter.toAggregate(orderPO, orderItemDao.selectByOrderId(orderId));
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void save(OrderAggregate aggregate) {
        // 主表和明细同一事务写入
        orderDao.insert(OrderPoConverter.toOrderPO(aggregate));
        orderItemDao.insertBatch(OrderPoConverter.toItemPOList(aggregate));
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void update(OrderAggregate aggregate) {
        // 条件更新：WHERE order_id = ? AND version = ?，0 行说明期间被并发修改
        if (orderDao.updateByOrderIdAndVersion(OrderPoConverter.toOrderPO(aggregate)) != 1) {
            throw new BusinessException(ErrorCode.UPDATE_ZERO.getCode(), "订单已被修改，请重试: " + aggregate.getOrder().getOrderId());
        }
        orderItemDao.deleteByOrderId(aggregate.getOrder().getOrderId());
        orderItemDao.insertBatch(OrderPoConverter.toItemPOList(aggregate));
    }
}
```

示例中的 `update` 用到了乐观锁版本号，需要 `OrderEntity` 和 `OrderPO` 带 `version` 字段；工程有其他并发控制方式时按工程写。

## 3. Port

实现类 `{Xxx}Port` 标注 `@Component`，是防腐层：协议、序列化、超时交给 `gateway/` 等技术包，Port 负责把外部的结果翻译成本领域的语言。

- 按外部能力命名（`IProductPort`），方法名表达业务含义（`queryProduct`），不叫 `post(url, body)`。
- 参数和返回值只用领域类型，不出现 DTO、`Map`、JSON 字符串、HTTP 状态码；不声明受检异常。
- 调用失败抛 `BusinessException`；"查不到"等正常结果用 `null` 或领域对象表达，并在注释写明。
- 只做转换和错误映射，不做业务判断。

### 3.1 外部状态：外部枚举 → 领域枚举

外部服务返回的状态（业务状态、响应码）不用字符串常量判断，也不原样透传，而是经过三种表示：

| 表示 | 所在层 | 示例 |
|------|--------|------|
| 报文原值 | infrastructure，远程 DTO 的 `String` 字段 | `"4"` |
| 外部枚举 `{External}{Xxx}Enum` | infrastructure，和远程 DTO 放在一起，只描述对方的含义 | `ProductCenterStatusEnum.PRE_SALE` |
| 领域枚举 `{Xxx}EnumVO` | domain，表达本领域的含义 | `ProductSaleStatusEnumVO.OFF_SALE` |

- **报文原值 → 外部枚举**：外部枚举的 `fromCode`，未知值或空值抛 `BusinessException`，不猜测含义。远程 DTO 的状态字段保持 `String`，这样未知值能得到明确的报错，也不依赖 JSON 库对枚举的反序列化配置。
- **外部枚举 → 领域枚举**：在适配器里完成。业务状态在 PortConverter 里转成领域枚举；响应码在 Port 里转成领域结果（返回对象、返回 `null`）或 `BusinessException`。
- **两边互不引用**：外部枚举不认识领域枚举，领域枚举也不保存外部编码；对方改编码、加状态，只改外部枚举和适配器里的映射。
- **映射不写 `default`**：用 `switch` 表达式覆盖外部枚举的全部常量，外部枚举新增常量时编译报错，强制决定它在本领域的含义。外部有、本领域没有的状态，在映射里显式归并（如"预售"归为"不可售"），不能漏掉。

入站消息（如支付回调、其他服务的 MQ 消息）里的外部状态同样处理，转换写在 trigger 的 ApiConverter 里。

### 3.2 示例

```java
// domain：端口与领域枚举
public interface IProductPort {
    /** 查询商品；不存在返回 null，远程调用失败抛 BusinessException */
    ProductVO queryProduct(String skuId);
}

@Getter
@AllArgsConstructor
public enum ProductSaleStatusEnumVO {
    ON_SALE("ON_SALE", "可售"),
    OFF_SALE("OFF_SALE", "不可售"),
    SOLD_OUT("SOLD_OUT", "售罄"),
    ;

    private final String code;
    private final String desc;
}

@Getter
@Builder
@EqualsAndHashCode
@AllArgsConstructor(access = AccessLevel.PRIVATE)
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ProductVO {
    private String skuId;
    private String productName;
    private MoneyVO price;
    private ProductSaleStatusEnumVO saleStatus;

    public boolean isOnSale() {
        return ProductSaleStatusEnumVO.ON_SALE == saleStatus;
    }
}
```

```java
// infrastructure：外部商品中心的枚举，和远程 DTO 放在一起
@Getter
@AllArgsConstructor
public enum ProductCenterStatusEnum {
    ON_SHELF("1", "上架"),
    OFF_SHELF("2", "下架"),
    SOLD_OUT("3", "售罄"),
    PRE_SALE("4", "预售"),
    ;

    private final String code;
    private final String desc;

    /** 未知或空值直接报错，不猜测含义 */
    public static ProductCenterStatusEnum fromCode(String code) {
        for (ProductCenterStatusEnum value : values()) {
            if (value.code.equals(code)) {
                return value;
            }
        }
        throw new BusinessException(ErrorCode.REMOTE_CALL_ERROR.getCode(), "未知的商品中心状态: " + code);
    }
}

@Getter
@AllArgsConstructor
public enum ProductCenterCodeEnum {
    SUCCESS("0000", "成功"),
    NOT_FOUND("0404", "商品不存在"),
    SYSTEM_ERROR("9999", "系统异常"),
    ;

    private final String code;
    private final String desc;

    // fromCode 写法同 ProductCenterStatusEnum
}

// infrastructure：远程 DTO 保持报文原样
@Data
@Builder
public class ProductRequestDTO {
    private String skuId;
}

@Data
public class ProductResponseDTO {
    private String code;          // 响应码，由 ProductCenterCodeEnum 解析
    private String info;
    private String skuId;
    private String productName;
    private Long price;           // 单位分
    private String currency;
    private String status;        // 商品状态，由 ProductCenterStatusEnum 解析
}
```

```java
// infrastructure：Port，响应码 → 领域结果
@Slf4j
@Component
public class ProductPort implements IProductPort {

    @Resource
    private ProductGatewayService productGatewayService;   // gateway/ 技术包，只收发 DTO

    @Override
    public ProductVO queryProduct(String skuId) {
        ProductResponseDTO response;
        try {
            response = productGatewayService.queryProduct(ProductPortConverter.toRequestDTO(skuId));
        } catch (Exception e) {
            // 超时、连接失败等客户端异常统一映射，不进入领域层
            log.error("商品服务调用异常 skuId={}", skuId, e);
            throw new BusinessException(ErrorCode.REMOTE_CALL_ERROR.getCode(), "商品服务调用失败");
        }
        if (response == null) {
            throw new BusinessException(ErrorCode.REMOTE_CALL_ERROR.getCode(), "商品服务无响应");
        }
        // 不写 default：外部新增响应码时编译报错，强制补上映射；未知编码已在 fromCode 中报错
        return switch (ProductCenterCodeEnum.fromCode(response.getCode())) {
            case SUCCESS -> ProductPortConverter.toProductVO(response);
            case NOT_FOUND -> null;
            case SYSTEM_ERROR -> throw new BusinessException(ErrorCode.REMOTE_CALL_ERROR.getCode(),
                    "商品服务返回失败: " + response.getInfo());
        };
    }
}

// infrastructure：PortConverter，外部枚举 → 领域枚举
public final class ProductPortConverter {

    private ProductPortConverter() {
    }

    public static ProductRequestDTO toRequestDTO(String skuId) {
        return ProductRequestDTO.builder().skuId(skuId).build();
    }

    public static ProductVO toProductVO(ProductResponseDTO response) {
        if (response == null) {
            return null;
        }
        return ProductVO.builder()
                .skuId(response.getSkuId())
                .productName(response.getProductName())
                .price(MoneyVO.of(response.getPrice(), response.getCurrency()))
                .saleStatus(toSaleStatus(ProductCenterStatusEnum.fromCode(response.getStatus())))
                .build();
    }

    /** 不写 default：外部枚举新增常量时编译报错，强制决定它在本领域的含义 */
    private static ProductSaleStatusEnumVO toSaleStatus(ProductCenterStatusEnum status) {
        return switch (status) {
            case ON_SHELF -> ProductSaleStatusEnumVO.ON_SALE;
            case OFF_SHELF, PRE_SALE -> ProductSaleStatusEnumVO.OFF_SALE;   // 预售期还不能下单，本领域视为不可售
            case SOLD_OUT -> ProductSaleStatusEnumVO.SOLD_OUT;
        };
    }
}
```

`ProductGatewayService`（`gateway/`）只负责地址、协议、序列化和超时，返回 `ProductResponseDTO`，不认识领域对象，也不解释响应码和状态。

## 4. 反模式

```java
// ❌ Repository 接口返回 PO、暴露 DAO、把影响行数交给领域层
OrderPO findById(Long id);

int updateStatus(String orderId, int status);
// ✅ 领域类型 + 业务语义，行数在实现内校验

// ❌ Repository 或 PO 里做业务判断
if(aggregate.getItems().size() >10){throw...;}
// ✅ 规则放在聚合或领域服务，Repository 只负责读写

// ❌ Port 接口泄漏远程结构，或吞掉异常返回 false：调用方分不清"不存在"和"服务不可用"
ProductResponseDTO queryProduct(String skuId);
catch(Exception e){ return null; }
// ✅ 返回领域对象；调用失败抛 BusinessException

// ❌ 外部状态用字符串常量判断，或原样透传进领域：领域里只能写 "1".equals(product.getStatus())
private static final String CODE_SUCCESS = "0000";
.status(response.getStatus())
// ✅ 先用外部枚举解析，再在适配器里映射成领域枚举

// ❌ 领域直接使用外部枚举；外部枚举上写 toDomain()；领域枚举里保存外部编码
// ✅ 外部枚举和领域枚举互不引用，映射只写在适配器

// ❌ 映射写 default 兜底：外部新增状态时被悄悄归到某个内部状态
default -> ProductSaleStatusEnumVO.ON_SALE;
// ✅ switch 覆盖外部枚举的全部常量，不写 default

// ❌ Port 读写 dao / redis（如为了缓存远程结果）
// ✅ 由领域服务编排：先查 Repository，未命中调 Port，再交给 Repository 写缓存

// ❌ 把配置中心、其他服务的接口包装成 IXxxRepository
// ✅ 数据归对方的一律走 Port
```
