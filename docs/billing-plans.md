# Proxlane 課金プラン仕様

## 1. 方針

Proxlane は OSS と Hosted SaaS の両方を想定する。課金対象は Hosted SaaS の Relay、Dashboard、運用、サポートであり、OSS としてセルフホストする場合は Proxlane 側の Stripe 課金対象にしない。

MVP では、Stripe の定額 subscription を使う。転送量の従量課金は初期実装では行わず、Proxlane 側の quota で制限する。これにより、想定外の高額請求、未回収、Stripe usage 同期の複雑さを避ける。

## 2. プラン一覧

価格は初期設定として Free / Plus / Pro の 3 段階で開始する。学生や個人開発者が入りやすい価格を優先し、チーム向けや Enterprise は後続フェーズで追加する。

| プラン | 月額 | 年額 | 対象 |
| --- | ---: | ---: | --- |
| Free | 0円 | 0円 | 個人開発、試用、OSS 体験 |
| Plus | 390円 | 未定 | 学生、個人開発者、継続利用 |
| Pro | 980円 | 未定 | フリーランス、小規模開発、本格利用 |

年額は約 2 か月分を割り引く想定とする。MVP では月額のみで開始し、年額は Stripe 運用に慣れてから追加してもよい。

## 3. 機能上限

| 項目 | Free | Plus | Pro |
| --- | ---: | ---: | ---: |
| seat 数 | 1 | 1 | 1 |
| active tunnel | 1 | 3 | 10 |
| Agent token | 1 | 3 | 10 |
| HTTP/HTTPS tunnel | yes | yes | yes |
| WebSocket | yes | yes | yes |
| TCP tunnel | yes | yes | yes |
| TCP/HTTP 上限 | 各1 | 各3 | 各10 |
| UDP tunnel | no | no | no |
| ランダム subdomain | yes | yes | yes |
| 予約 subdomain | 0 | 1 | 5 |
| 固定 TCP port | 0 | 0 | 後で実装 |
| custom domain | 0 | 0 | 後で実装 |
| 月間転送量 | 5GB | 50GB | 200GB |
| 同時 connection 目安 | 20 | 100 | 300 |
| connection log 保持 | 24時間 | 7日 | 14日 |
| access control | basic only | Basic / Bearer | Basic / Bearer / IP allowlist |
| request inspection | limited | limited | yes |
| support | community | email | email |

## 4. quota の扱い

MVP では超過課金をしない。上限到達時の挙動は次の通り。

| quota | 80% 到達 | 100% 到達 |
| --- | --- | --- |
| 月間転送量 | Dashboard に警告を表示 | 新規 tunnel 作成を拒否し、既存 tunnel は段階的に制限 |
| active tunnel | warning なし | 新規 tunnel 作成を拒否 |
| Agent token | warning なし | 新規 token 作成を拒否 |
| 同時 connection | Relay log に warning | 新規 connection を `429` または connection refused |

有料プランでは、初期運用に限り 100% 到達後も短い grace を設けることができる。ただし自動で追加課金しない。

現時点の Relay 実装では、active tunnel、TCP/HTTP 各上限、月間転送量、予約 subdomain 所有を「新規 tunnel 作成時」に判定する。既存 tunnel の途中切断、connection 単位の制限、grace period は後続フェーズで実装する。

## 5. Stripe 設計

### 5.1 Products

Stripe Dashboard には次の Product を作る。

| Product | 用途 |
| --- | --- |
| `Proxlane Plus` | 個人向け軽量有料プラン |
| `Proxlane Pro` | 個人向け有料プラン |

Free は Stripe Product を作らず、Proxlane DB 上の default plan として扱う。

### 5.2 Prices

MVP では monthly price を先に作成する。

| 環境変数 | Stripe Price | 金額 | interval |
| --- | --- | ---: | --- |
| `STRIPE_PRICE_PLUS_MONTHLY` | Plus monthly | 390円 | month |
| `STRIPE_PRICE_PRO_MONTHLY` | Pro monthly | 980円 | month |
| `STRIPE_PRICE_PLUS_YEARLY` | Plus yearly | 未定 | year |
| `STRIPE_PRICE_PRO_YEARLY` | Pro yearly | 未定 | year |

年額 price は作成しても、MVP 画面では非表示にしてよい。

### 5.3 metadata

Stripe Product / Price には次の metadata を設定する。

| key | 値の例 | 用途 |
| --- | --- | --- |
| `proxlane_plan` | `pro` | Proxlane の plan 判定 |
| `proxlane_interval` | `month` | 月額/年額の識別 |
| `limits_version` | `v1` | quota 定義の世代管理 |

Proxlane 側では price id を信用し、client から渡された plan 名だけでプラン変更しない。

## 6. Webhook 同期

受け取る Stripe event:

| event | 処理 |
| --- | --- |
| `checkout.session.completed` | `stripeCustomerId` と `stripeSubscriptionId` を紐づける |
| `customer.subscription.created` | plan / status / currentPeriodEnd を同期 |
| `customer.subscription.updated` | plan / status / currentPeriodEnd を更新 |
| `customer.subscription.deleted` | plan を `free` に戻す、または期間終了まで entitlement を維持 |
| `invoice.payment_failed` | `past_due` として grace / 制限の判定に使う |
| `invoice.paid` | `active` へ回復した場合に制限を解除 |

Webhook は raw body と `Stripe-Signature` を使って署名検証してから処理する。検証前に JSON parse しない。

Production の webhook endpoint は `https://proxlane.com/api/billing/webhook` を使う。設定手順と env の置き場所は [運用ノート](operations.md) に残す。

## 7. status と entitlement

| Stripe status | Proxlane entitlement |
| --- | --- |
| `trialing` | 対象 plan を有効 |
| `active` | 対象 plan を有効 |
| `past_due` | grace 中は有効、期限後は Free 相当に制限 |
| `unpaid` | Free 相当に制限 |
| `canceled` | Free 相当に制限 |
| `incomplete` | Free のまま |
| `incomplete_expired` | Free のまま |

MVP では free trial を設けない。Free plan が試用枠を兼ねる。

## 8. Dashboard 表示

`/app/billing` では次を表示する。

- 現在の plan
- subscription status
- 月間転送量と上限
- active tunnel / token / reserved endpoint の使用数
- plan 比較表
- Upgrade button
- Customer Portal button

Checkout から戻った直後は Webhook 反映前の可能性があるため、Dashboard は短時間 polling するか、手動 refresh 導線を置く。

## 9. 将来検討

- Stripe Meters API による usage-based billing
- 追加転送量パック
- 追加 seat
- dedicated Relay add-on
- annual plan の公開
- Stripe Tax
- 請求書払い
- Enterprise SLA

## 10. 参考

2026-05-17 時点で、Stripe は subscription、Checkout、Customer Portal、usage-based billing、Webhook signature verification を公式機能として提供している。Proxlane MVP では Checkout / Customer Portal / Webhook を使い、usage-based billing は後続フェーズに回す。

- Stripe Billing Pricing: https://stripe.com/billing/pricing
- Stripe Checkout: https://stripe.com/payments/checkout
- Stripe Customer Portal: https://docs.stripe.com/billing/subscriptions/integrating-customer-portal
- Stripe Subscriptions: https://docs.stripe.com/payments/subscriptions
- Stripe Webhook Signatures: https://docs.stripe.com/webhooks/signatures
