* Oracle公式
https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html?utm_source=chatgpt.com
* 実装方法
https://www.baeldung.com/java-virtual-thread-vs-thread?utm_source=chatgpt.com


# Java Virtual Thread / Blocking I/O / CompletableFuture キャッチアップまとめ

## 1. Blocking I/O Operation とは？

### I/Oとは
外部とのやり取り全般。

例：

- DBアクセス
- HTTP API呼び出し
- ファイル読み書き
- Redis
- Socket通信

### Blocking I/O

**I/O完了待ちの間、スレッドが停止（占有）される処理**

例：

```java
String response = httpClient.get("https://example.com");
```

レスポンスが返るまで thread が待機する。

イメージ：

```text
Thread A:
DB問い合わせ
↓
待つ（BLOCKING）
↓
結果受け取る
```

---

## 2. Non-blocking I/O

待機中に thread を占有しない。

イメージ：

```text
注文
↓
「できたら呼んで」
↓
別作業
↓
完成通知
↓
受け取り
```

例：

```java
CompletableFuture<Response> future =
    client.sendAsync(request);

future.thenAccept(response -> {
    System.out.println(response.body());
});
```

---

# 3. Virtual Thread は「速い」のではなく「スケールする」

公式：

> Virtual threads are not faster threads;
> they exist to provide scale (higher throughput),
> not speed (lower latency).

### Speed（Latency）

**1件の処理が終わる速さ**

例：

```text
DB応答 = 100ms
```

なら：

```text
Platform Thread → 100ms
Virtual Thread → 100ms
```

速くはならない。

---

### Scale（Throughput）

**単位時間あたりに処理できる量**

例：

### Platform Thread

```text
200 thread pool

180個 → DB待ち
20個 → 実行中

新規 request → thread空きなし
```

thread 数がボトルネック。

---

### Virtual Thread

```text
100000 virtual thread

95000 → 待機
5000 → 実行
```

でも必要な OS thread（carrier thread）は：

```text
CPU core数 × 少し
```

程度。

### 結果

```text
throughput ↑
concurrency ↑
resource efficiency ↑
```

---

# 4. Spring MVC の従来モデル

Spring MVC はもともと：

## thread-per-request モデル

```text
request
↓
Tomcat worker thread 確保
↓
Controller
↓
Service
↓
Repository
↓
DB/API待ち（blocking）
↓
response
↓
thread返却
```

### 問題

DB待ち中も thread を占有。

```text
maxThreads = 200
```

なら：

```text
200 request が実質上限
```

---

# 5. CompletableFuture 時代

スケールさせるために：

```java
@GetMapping
public CompletableFuture<User> getUser() {
    return CompletableFuture.supplyAsync(
        () -> service.getUser()
    );
}
```

### 動き

```text
request
↓
Tomcat thread
↓
CompletableFuture返却
↓
Tomcat thread解放
↓
別thread poolで処理
↓
response
```

### 注意

内部が blocking なら：

```java
jdbcTemplate.query(...)
```

worker thread が block する。

つまり：

```text
Tomcat thread節約
≠
完全 non-blocking
```

---

# 6. WebFlux / Reactor の世界

### 特徴

**request ごとに thread を持たない**

```text
few event loop threads
↓
大量 request を multiplex
```

例：

```java
@GetMapping("/user")
public Mono<User> getUser() {
    return webClient.get()
        .retrieve()
        .bodyToMono(User.class);
}
```

### 内部

```text
event loop thread
↓
HTTP call
↓
OSへI/O登録
↓
thread解放
↓
response通知
↓
callback再開
```

### メリット

```text
高スケール
少数 thread
```

### デメリット

```text
flatMap()
map()
zip()
```

チェーンが複雑化。

---

# 7. Virtual Thread 時代

## Spring MVC + Virtual Thread

設定：

```properties
spring.threads.virtual.enabled=true
```

すると：

```text
Platform Thread
↓
Virtual Thread
```

に置き換わる。

### 普通の blocking code が使える

```java
@GetMapping("/user")
public User getUser() {
    return service.getUser();
}
```

---

## WebClient + block() も現実的

昔：

```java
.block()
```

は悪とされた。

理由：

```text
platform thread を占有
```

するから。

---

### Virtual Thread 時代

```java
public User getUser() {
    return webClient.get()
        .uri("/user")
        .retrieve()
        .bodyToMono(User.class)
        .block();
}
```

でも OK なケースが増えた。

理由：

```text
block()
↓
virtual thread park
↓
carrier thread解放
↓
responseでresume
```

---

# 8. Virtual Thread の内部

## 構造

```text
Virtual Thread
↓ mount
Carrier Thread(OS Thread)
↓
実行
```

実行中だけ OS thread に乗る。

---

## 待機時

```text
before blocking
↓
park virtual thread
↓
OSにI/O登録
↓
carrier thread解放
↓
response
↓
resume
```

### ポイント

**thread を待たせるのではなく**

```text
「データ来たら教えて」
```

を OS に依頼する。

---

# 9. 「OSにI/O登録」とは？

OSのイベント通知機構を使う。

### Linux

`epoll`

### macOS

`kqueue`

### Windows

`IOCP`

### イメージ

昔：

```text
店員：
ラーメンできるまで厨房前で待つ
```

今：

```text
店員：
「できたらベル鳴らして」
↓
別作業
↓
ベル鳴る
↓
再開
```

OS がベル係。

---

# 10. 全ての待機で thread 解放される？

**NO**

## 解放される（park可能）

- socket wait
- HTTP wait
- DB wait
- sleep
- queue wait
- lock wait

---

## 解放されない

### CPU-bound

```java
while(true) {
    heavyCalculation();
}
```

CPU占有。

---

### Pinning

例：

```java
synchronized(lock) {
    Thread.sleep(1000);
}
```

`synchronized` 中に block すると：

```text
carrier thread を解放できない
```

場合がある。

これを：

## Pinning（ピニング）

と呼ぶ。

### 対策

Loom時代は：

```java
ReentrantLock
```

が推奨寄り。

---

# 11. CompletableFuture vs Virtual Thread

## CompletableFuture

### 人間が non-blocking を書く

```java
CompletableFuture.allOf(...)
    .thenApply(...)
```

メリット：

```text
高効率
fan-out/fan-in
```

デメリット：

```text
複雑
例外処理が辛い
可読性低下
```

---

## Virtual Thread

### JVM が non-blocking を面倒見る

普通に書く：

```java
User user = api.get();
Order order = repo.find();
```

内部だけ：

```text
park / resume
```

してくれる。

---

## 現在の基本戦略（Java 21+）

まず：

```text
Spring MVC
+
Virtual Thread
+
blocking code
```

を検討。

必要なら：

```text
CompletableFuture
Reactor
WebFlux
```

を使う。

---

# 12. Virtual Thread の現実

thread制約は減るが、

## 外部リソース制約は残る

例：

```text
Virtual Thread 10000本
↓
HikariCP = 20 connection
↓
DB待ち大渋滞
```

つまり：

```text
thread bottleneck は消える
resource bottleneck は消えない
```
