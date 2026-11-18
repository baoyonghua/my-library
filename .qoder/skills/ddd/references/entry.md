# 入口层：Case 与 Trigger

Trigger 接收 HTTP / MQ / Job 输入，经 ApiConverter 转换后调用领域服务或 Case；Case 是可选层，只做跨领域编排。示例省略 package 和 import。

## 1. Case

### 1.1 什么时候需要

| 情况 | 做法 |
|------|------|
| 单领域业务，即使步骤多 | Trigger 直接调领域服务，步骤属于该领域的领域服务 |
| 需要依次调用多个领域的服务 | 新建 Case |
| 多个入口（HTTP、MQ、Job）要执行同一段跨域编排 | 新建 Case 供各入口复用 |
| 已有 Trigger 注入多个领域服务并串联调用、出现业务分支 | **先询问用户**是否重构为 Case，说明信号和打算迁移的代码，不擅自重构 |

### 1.2 职责与禁止

| Case 负责 | Case 禁止 |
|-----------|-----------|
| 决定调用哪些领域服务、按什么顺序 | 注入 Repository、Port、DAO、客户端 |
| 跨域数据转换，经 `{Xxx}CaseConverter` | 加 `@Transactional`，或用一个事务包住多个领域 |
| 让异常原样抛出 | 捕获异常后返回 null / false |
| 保证整个 Case 可以安全地重复调用 | 写业务规则（计算金额、判断状态） |

- 包：`cases/{主导领域}/{capability}/`，放 `I{Xxx}Case`、`{Xxx}Case`、`converter/`。case 只依赖 domain 和 types，不接收 api DTO，不返回 `Response<T>`。
- 跨域只能最终一致：每个领域的写入是该领域 Repository 内的一个本地事务，靠"每一步幂等 + 重试 / 补偿"保证最终一致。

### 1.3 示例：支付成功后更新订单并发放积分

```java
public interface IPaySuccessCase {
    /** 支付成功后的订单更新与积分发放；可以重复调用，结果不变 */
    void handle(String orderId, String tradeNo);
}

@Slf4j
@Service
public class PaySuccessCase implements IPaySuccessCase {

    @Resource
    private IOrderPayService orderPayService;        // 订单领域
    @Resource
    private ICreditGrantService creditGrantService;  // 积分领域

    @Override   // 不加 @Transactional：两步分别在各自的 Repository 内提交
    public void handle(String orderId, String tradeNo) {
        // 1. 订单领域：状态改为已支付（事务 1，按 orderId 幂等，重复调用返回已有结果）
        OrderEntity order = orderPayService.paySuccess(orderId, tradeNo);
        // 2. 订单领域结果 → 积分领域命令；积分数由积分领域计算，这里只传事实
        CreditGrantCommand command = PaySuccessCaseConverter.toCreditGrantCommand(order);
        // 3. 积分领域：发放积分（事务 2，按 bizId 幂等）。失败直接抛出，由入口重试
        creditGrantService.grant(command);
    }
}
```

事务 1 提交后若第 3 步失败，异常抛到 MQ 监听器，broker 重投；重投时第 1 步因幂等直接返回，第 3 步得到重试，两个领域最终一致。入口是 HTTP 这类没有自动重投的调用时，由补偿 Job 扫描"已支付、未发积分"的记录重新调用 Case。

## 2. Trigger

| 类型 | 位置 | 说明 |
|------|------|------|
| HTTP | `trigger/http/{Xxx}Controller` | 实现 api 模块的 `I{Capability}Service` |
| 消息 / 事件 | `trigger/listener/{Xxx}Listener` | 一个类处理一类消息 |
| 定时任务 | `trigger/job/{Xxx}Job` | 定时扫描，分批逐笔调用 |
| 转换 | `trigger/converter/{Xxx}ApiConverter` | 见 [converter.md](converter.md) |

职责：

1. 接收输入，打印入口日志（只记关键 ID，不打印敏感字段）。
2. 基础参数校验（必填、格式、长度），按工程约定（如 `@Valid`）；"订单能不能取消"这类业务规则留给领域。
3. 经 ApiConverter 把 DTO 转成命令（只需一两个字段时直接取值传参），调用 Case 或领域服务。
4. 结果经 ApiConverter 转成 ResponseDTO，不直接返回领域对象。

### 2.1 Controller

```java
@Slf4j
@RestController
@RequestMapping("/order")
public class OrderController implements IOrderApiService {

    @Resource
    private IOrderService orderService;

    @Override
    @PostMapping("/create")
    public Response<CreateOrderResponseDTO> createOrder(@RequestBody @Valid CreateOrderRequestDTO request) {
        log.info("创建订单 userId:{} skuId:{}", request.getUserId(), request.getSkuId());
        OrderEntity order = orderService.createOrder(OrderApiConverter.toCommand(request));
        return Response.success(OrderApiConverter.toResponseDTO(order));
    }
}
```

工程已有全局异常处理（如 app 的 `GlobalExceptionAdvice`）时，Controller 不捕获异常，`BusinessException` 由它转换为 `Response`。没有统一处理时才在 Controller 中捕获，非业务异常返回 `ErrorCode.UN_ERROR`，不把 `e.getMessage()` 返回给调用方。

### 2.2 MQ Listener

```java
@Slf4j
@Component
public class PaySuccessListener {

    @Resource
    private IPaySuccessCase paySuccessCase;

    // 按项目使用的 MQ 客户端注册监听
    public void onMessage(PaySuccessMessageDTO message) {
        log.info("收到支付成功消息 orderId:{}", message.getOrderId());
        try {
            paySuccessCase.handle(message.getOrderId(), message.getTradeNo());
        } catch (Exception e) {
            log.error("支付成功消息处理失败 orderId:{}", message.getOrderId(), e);
            throw e;   // 继续抛出，由 broker 重投或转入死信
        }
    }
}
```

- 同一条消息可能投递多次，处理逻辑必须幂等；幂等由领域和 Repository 保证（唯一键、条件更新），监听器不自己查库去重。
- Job 可以捕获单笔异常并记录，前提是失败的数据下一轮还会被扫到；不能影响同批其他数据。

## 3. 反模式

```java
// ❌ Controller 里写业务逻辑：换成 MQ 入口就得再写一遍
if (order.getStatus() != OrderStatusEnumVO.CREATE) { return Response.fail(...); }
// ✅ 规则放在实体 / 领域服务，Controller 只转换和调用

// ❌ MQ 监听器或 Case 吞掉异常：broker 认为消费成功，不再重投，数据永久不一致
catch (Exception e) { log.error("发放积分失败", e); }
// ✅ 记录日志后继续抛出

// ❌ 把 api DTO 传进 Case 或领域服务：外部契约一改，领域就跟着改
void handle(PaySuccessMessageDTO message);
// ✅ Trigger 负责转换，Case 和领域只接收命令或简单值

// ❌ Trigger 注入 Repository / DAO，或开事务
// ✅ 只注入领域服务或 Case 接口
```
