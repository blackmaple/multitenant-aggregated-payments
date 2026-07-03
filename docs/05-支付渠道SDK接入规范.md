# 《05-支付渠道SDK接入规范.md》

## 1. 设计目标
通过统一 SDK 接口屏蔽渠道差异，业务方只需理解统一模型。

## 2. 包结构
- `MultiTenant.Payments.Abstractions`
- `MultiTenant.Payments.Core`
- `MultiTenant.Payments.Channels.Wechat`
- `MultiTenant.Payments.Channels.Alipay`
- `MultiTenant.Payments.Channels.Stripe`

## 3. 核心接口定义（C#）

```csharp
public interface IPaymentProvider
{
    Task<CreateOrderResponse> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct = default);
    Task<QueryOrderResponse> QueryOrderAsync(QueryOrderRequest request, CancellationToken ct = default);
    Task<CloseOrderResponse> CloseOrderAsync(CloseOrderRequest request, CancellationToken ct = default);
}

public interface IRefundProvider
{
    Task<CreateRefundResponse> CreateRefundAsync(CreateRefundRequest request, CancellationToken ct = default);
    Task<QueryRefundResponse> QueryRefundAsync(QueryRefundRequest request, CancellationToken ct = default);
}

public interface INotifyHandler
{
    Task<NotifyParseResult> ParseAndVerifyAsync(string payload, IDictionary<string, string> headers, CancellationToken ct = default);
}
```

## 4. 统一数据模型（示例）

```csharp
public sealed record CreateOrderRequest(
    long TenantId,
    string AppId,
    string MerchantOrderNo,
    decimal Amount,
    string Currency,
    string ChannelCode,
    string Subject,
    string NotifyUrl,
    IDictionary<string, string>? Metadata
);

public sealed record CreateOrderResponse(
    string PlatformOrderNo,
    string? ChannelOrderNo,
    string Status,
    string? PayUrl,
    string? QrCode,
    string? PrepayInfo,
    string RawPayload
);
```

## 5. 签名与安全规范
- 请求必须包含 timestamp + nonce + signature
- 统一签名串规则：按字段字典序拼接
- 私钥仅存储于密钥管理系统（KMS/Vault）
- SDK 内部严禁输出明文密钥到日志

## 6. 异常码
- PAY001 参数非法
- PAY002 签名验证失败
- PAY003 渠道超时
- PAY004 渠道不可用
- PAY005 幂等冲突
- PAY006 订单状态非法
- PAY999 未知异常

## 7. 幂等规范
- 下单请求必须携带 `IdempotencyKey`
- 平台以 `(TenantId + MerchantOrderNo + IdempotencyKey)` 做幂等判定
- 幂等命中返回首次成功结果快照

## 8. 回调处理规范
1. 读取原始 body（不可二次序列化）
2. 验签
3. 幂等去重（eventId）
4. 更新订单状态（状态机校验）
5. 投递支付成功事件
6. 返回渠道要求的成功应答
