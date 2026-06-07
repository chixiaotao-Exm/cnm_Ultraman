# PayPal BA 链提取流程原理
苹果浏览器填表  https://github.com/chixiaotao-Exm/apple-app-links

地址API  https://api.chixiaotao.cn/api/jp

paypal提链原理 https://github.com/chixiaotao-Exm/cnm_Ultraman

paypal提链https://api.chixiaotao.cn/paypal

买代理的网站（不是我的,和我没关系,仅推荐）

https://pay.ldxp.cn/shop/xiaodaidai.IP

## 最终正确流程

```text
1. checkout 阶段：JP 代理
2. provider / Stripe 阶段：US 代理
3. approve 阶段：JP 代理
4. billing_details：US 账单地址
```

## 分阶段原因

### 1. checkout 阶段使用 JP

先通过 ChatGPT 后端创建 0 元 promo checkout：

```text
ChatGPT backend -> create checkout -> 得到 cs_live
```

这一阶段使用 JP 出口，能稳定创建 checkout session。

但 checkout 的国家和币种不能改成 JP/JPY。之前测试过：

```text
country=JP currency=JPY
```

会导致 Stripe confirm 报：

```text
payment_method_types_mismatch
```

原因是当前这个 Stripe Checkout Session 的 PayPal 支付方式按 `US/USD` 可用。

因此正确状态是：

```text
checkout proxy = JP
checkout country = US
checkout currency = USD
```

### 2. provider / Stripe 阶段使用 US

拿到 `cs_live` 后，切换到 US 代理执行 Stripe provider 阶段：

```text
Stripe init
create PayPal payment_method
confirm payment page
```

这一段需要更像美国环境，否则 PayPal payment method / setup intent 容易不匹配。

成功后 confirm 返回：

```text
submission_attempt.state = requires_approval
```

含义是：Stripe 已经接受 PayPal payment method，但这个 checkout 还需要 ChatGPT 后端 approve。

### 3. approve 阶段切回 JP

`requires_approval` 后调用：

```text
https://chatgpt.com/backend-api/payments/checkout/approve
```

这一阶段切回 JP 代理。

成功时返回：

```text
result = approved
```

日志表现为：

```text
approve: approved
```

随后继续轮询 Stripe payment page，等待 PayPal BA / redirect 链生成。

## 真正的失败原因

这次调试 JSON 里看到的核心错误是：

```text
setup_intent.last_setup_error.code = setup_attempt_failed
decline_code = generic_decline
payment_method.type = paypal
billing_details.address.country = JP
```

也就是说，当时状态是：

```text
checkout = US/USD
provider = US
payment_method billing_details = JP
```

Stripe 最后拒绝了这个 PayPal setup intent。

所以关键修复是：

```text
PayPal payment_method 的 billing_details 改成 US 地址
```

修复后日志会显示：

```text
provider: billing country=US
```

## 成功原理总结

最终跑通的关键组合是：

```text
JP 创建 ChatGPT checkout
US 创建 Stripe / PayPal payment method 并 confirm
JP 调 ChatGPT approve
US 账单地址匹配 US/USD checkout
Stripe setup_intent 不再 generic_decline
最后拿到 PayPal approve / BA 链
```

之前失败的核心是：

```text
US checkout + JP billing
```

修正后是：

```text
US checkout + US billing + US provider + JP approve
```

