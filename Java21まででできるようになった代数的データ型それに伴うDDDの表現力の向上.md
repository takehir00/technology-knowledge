# JavaでADT（代数的データ型）が使えるようになった話

## 結論

> Java 21で「完全なADT」が導入されたわけではない。

ただし：

- `sealed`
- `record`
- pattern matching (`switch`)

の組み合わせにより、

> **ADT（代数的データ型）っぽく書けるようになった**

が正確。

つまり、

> **Java 17〜21で段階的にADTを構成するピースが揃った**

という理解が良い。

---

## ADT（代数的データ型）とは？

ざっくり言うと：

> **型で状態のバリエーションを安全に表現する仕組み**

例（Rust/Kotlinっぽいイメージ）：

```text
Payment =
    CreditCard(cardNo)
  | BankTransfer(account)
  | Cash
```

「Paymentはこの3種類のどれか」を型で表現する。

---

## Java 21以前の課題

昔のJavaでは：

```java
interface Payment {}

class CreditCard implements Payment {}

class BankTransfer implements Payment {}

class Cash implements Payment {}
```

問題：

- 継承先が自由
- 網羅性チェックがない
- `instanceof` 地獄
- `switch` が弱い

---

## 1. `sealed`（Java 17）

継承先を限定できる。

```java
sealed interface Payment
    permits CreditCard,
            BankTransfer,
            Cash {}
```

これにより：

> Paymentはこの3種類しか存在しない

ことをコンパイラが理解できる。

---

## 2. `record`（Java 16）

variant（状態）を簡潔に表現。

```java
record CreditCard(
    String cardNo
) implements Payment {}

record BankTransfer(
    String account
) implements Payment {}

record Cash()
    implements Payment {}
```

ここで：

> **sum type（和型）っぽさ**

が出る。

---

## 3. Pattern Matching for `switch`（Java 21）

最後のピース。

### 昔

```java
if (payment instanceof CreditCard cc) {
    processCard(cc.cardNo());
} else if (...) {
}
```

### 今

```java
return switch (payment) {

    case CreditCard(var cardNo)
        -> processCard(cardNo);

    case BankTransfer(var account)
        -> processTransfer(account);

    case Cash()
        -> processCash();
};
```

### 重要ポイント

`sealed` により：

> exhaustive check（網羅性チェック）

が効く。

例えば：

```java
sealed interface Payment
    permits CreditCard,
            BankTransfer,
            Cash,
            Crypto {}
```

を追加すると、

未対応の `switch` が：

> **コンパイルエラー**

になる。

これはRust/Kotlin/Haskell系のADTにかなり近い体験。

---

## 型理論的に見ると

```text
record  → product type（積）
sealed  → sum type（和）
pattern matching → 分解
```

この3つが合体し、

> **Javaでも代数的データ型的モデリングが可能**

になった。

---

# Java21+でDDD設計はどう変わる？

## 結論

> DDDそのものは変わらない。

ただし、

> **実装スタイルがかなり変わる**

特に：

- Value Object
- Domain Event
- State表現
- Result型

が強くなる。

逆に、

> **Entityはあまり変わらない**

---

## Value Object → `record` が超相性良い

### 従来

```java
public final class EmailAddress {

    private final String value;

    public EmailAddress(String value) {

        if (!isValid(value)) {
            throw new IllegalArgumentException();
        }

        this.value = value;
    }

    public String value() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        ...
    }

    @Override
    public int hashCode() {
        ...
    }
}
```

### 今

```java
public record EmailAddress(
    String value
) {

    public EmailAddress {

        if (!isValid(value)) {
            throw new IllegalArgumentException();
        }
    }
}
```

### 理由

Value Objectの特徴：

- immutable
- equality based
- identityなし

これはrecordの思想とほぼ一致。

### 今の推奨

```java
record Money(
    BigDecimal amount,
    Currency currency
)

record UserId(UUID value)

record OrderNumber(String value)
```

> **VOはrecord第一候補**

になりつつある。

---

## Entity → 意外と変わらない

Entityは：

> **identity + mutable lifecycle**

を持つ。

そのためrecordとは相性が悪い。

### 推奨

```java
class Order {

    private OrderId id;
    private OrderStatus status;

    void cancel() {
        ...
    }
}
```

### 原則

- Entity → class
- Value Object → record

の棲み分け。

record化すると：

> ドメインロジックが薄いDTO化

しやすいので注意。

---

## State表現がかなり変わる（重要）

### 従来

```java
enum PaymentStatus {
    CREATED,
    PAID,
    FAILED
}
```

問題：

状態ごとのデータを持ちにくい。

例えば：

```java
FAILED(reason)
```

を表現しにくい。

結果：

```java
String failureReason;
```

のようなnullable field増殖が起きがち。

---

### Java21+

```java
sealed interface PaymentState
    permits Created,
            Paid,
            Failed {}

record Created()
    implements PaymentState {}

record Paid(
    Instant paidAt
) implements PaymentState {}

record Failed(
    String reason
) implements PaymentState {}
```

Entity：

```java
class Payment {

    PaymentState state;
}
```

状態遷移：

```java
switch (state) {

    case Created c -> ...

    case Paid p -> ...

    case Failed f -> ...
}
```

### メリット

- nullable field削減
- 「この状態ならこの情報がある」を型保証
- State patternを型で表現可能

---

## Domain Event → `record` が超強い

### 従来

```java
class OrderCreatedEvent {

    private final OrderId orderId;
    private final Instant occurredAt;
}
```

### 今

```java
public record OrderCreated(
    OrderId orderId,
    Instant occurredAt
) {}
```

理由：

- immutable
- データ中心
- equality useful

なのでrecordとの相性が良い。

---

## Result型（失敗を型で表現）

### 従来

例外中心：

```java
throw new ValidationException();
```

### 今

```java
sealed interface CreateOrderResult
    permits Success,
            ValidationError {}

record Success(
    OrderId id
) implements CreateOrderResult {}

record ValidationError(
    String message
) implements CreateOrderResult {}
```

利用側：

```java
return switch (result) {

    case Success s
        -> ok(s.id());

    case ValidationError e
        -> ng(e.message());
};
```

### メリット

> exceptionではなく型で失敗を表現

Rust/Kotlin寄りの設計が可能。

---

## ただし注意：関数型に寄せ過ぎる罠

ADTが便利すぎると：

```java
switch(order) {
    ...
}
```

だらけになり、

> **貧血ドメインモデル（Anemic Domain Model）**

化しやすい。

本来：

```java
order.cancel();
```

であるべきロジックが、

```java
OrderService.cancel(order);
```

だらけになる。

---

## Java21+ × DDD のおすすめバランス

### Entity

→ OOのまま

```java
order.cancel();
```

### Value Object

→ `record`

### Domain Event

→ `record`

### State表現

→ `sealed + record`

### 分岐

→ pattern matching

---

## 一言まとめ

Java 21+は、

> 「エンタープライズOO全振りJava」

から、

> **「OO + 型安全な関数型のハイブリッドJava」**

へ進化している。
