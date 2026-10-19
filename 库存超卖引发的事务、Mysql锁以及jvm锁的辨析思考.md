# 库存超卖引发的事务、Mysql锁以及jvm锁的辨析思考

首先搞清楚：

1、事务机制

2、Mysql锁，（原来乐观锁是在mysql锁的基础上才成立的）

3、jvm锁（如synchronized）

## 处理库存超卖的场景

1、事务加乐观锁场景--此时事务在乐观锁外面但并不会有并发问题

~~~java
@Service
public class OrderService {
    
    @Autowired
    private ProductMapper productMapper;
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Transactional(rollbackFor = Exception.class)
    public void createOrder(Long productId, int quantity) {
        // 1. 查询库存和版本号
        Product product = productMapper.selectById(productId);
        if (product.getStock() < quantity) {
            throw new RuntimeException("库存不足");
        }
        
        // 2. 乐观锁扣库存
        int rows = productMapper.deductStock(productId, quantity, product.getVersion());
        if (rows == 0) {
            throw new RuntimeException("并发冲突，请重试");
        }
        
        // 3. 创建订单
        Order order = new Order();
        order.setProductId(productId);
        order.setQuantity(quantity);
        orderMapper.insert(order);
        
        // 任何一步失败，整个事务回滚，库存恢复
    }
}
~~~

2、事务加synchronized场景--事务在锁外面会导致并发问题，想要解决需要将锁放在事务外面

看这个经典错误写法：

```java
@Transactional
public void createOrder(Long productId, int quantity) {
    synchronized (this) {  // ← 锁在事务内部
        Product product = productMapper.selectById(productId);
        if (product.getStock() >= quantity) {
            productMapper.deductStock(productId, quantity);
        }
    }
    // synchronized 代码块结束 → 锁释放
    // 但事务还没提交！
}
```

**问题出在：synchronized 代码块执行完，锁就释放了，但此时事务还没提交。**

```bash
线程A：
  进入 synchronized → 扣库存 → 退出 synchronized（锁释放）
  → 但事务还没提交（还在 @Transactional 方法里）
  → 数据库里 stock 还是旧值（其他事务看不到未提交的修改）

线程B：
  进入 synchronized（锁已释放，可以进来）
  → 读到 stock 还是旧值（因为A没提交）
  → 也扣库存
  → 超卖！
```



**核心矛盾：**

- `synchronized` 锁的是 **Java 代码块**
- 事务的提交发生在**方法执行完之后**
- 锁释放了，事务没提交，中间有个“空窗期”，其他线程就能钻进来

## 场景梳理

我们来梳理一个场景，首先更新操作一定会有行锁，然后这个行锁保证和事务是一起提交的？ 此时我用了synchronized放在了事务的里面，此时锁释放，事务还未提交，线程b过来执行业务，虽然它读到了旧值，但线程a此时的行锁还没释放啊？线程b不依然不能更新吗？ 奥，线程b这一步虽然被阻塞了，但那个==读操作已近被线程b判断过了==,此时线程a事务提交，线程b依然要更新扣减操作，并发问题出现了。 如果jvm锁放在了事务外面就好了，保证在事务释放之前别的线程进不去，也就能保证查询操作的判断库存那一步操作没有被提前另一个线程执行，也就能防止库存超卖了。 
1、是不是因为查询操作本身的执行次数比较多导致业务耗时过长，才会没有行锁机制，不然给查操作也加上行锁，是不是就不会出现事务在jvm锁里面还是外面的问题了？
2、 乐观锁本质也依赖于更新操作的行锁机制对吧？不然同一时间都可以判断并发照样也会出现并发问题

### 一、你的场景梳理：完全正确

```
线程A：
  进入 synchronized
  读到 stock = 1
  判断 1 >= 1，通过
  UPDATE stock = 1 - 1 = 0  → 加行锁
  退出 synchronized（JVM锁释放）
  → 但事务还没提交，行锁还在

线程B：
  进入 synchronized（JVM锁已释放，可以进）
  读到 stock = 1          ← 读到旧值，因为A没提交
  判断 1 >= 1，通过       ← 判断已经完成
  UPDATE stock = 1 - 1 = 0
  → 被A的行锁阻塞，等待

线程A：
  COMMIT  → 行锁释放

线程B：
  阻塞结束，继续执行 UPDATE
  → stock = 1 - 1 = 0（基于它读到的旧值1算的）
  → 超卖！
```



**核心：行锁只保证“更新不冲突”，但B的判断已经基于旧值完成了，行锁拦不住“已经通过判断”的更新。**

你说得完全对。

------

### 二、如果把 JVM 锁放在事务外面

```
线程A：
  进入 synchronized
  开启事务
  读到 stock = 1
  判断通过
  UPDATE  → 加行锁
  事务提交 → 行锁释放
  退出 synchronized（JVM锁释放）

线程B：
  进入 synchronized（此时A已经全部完成）
  开启事务
  读到 stock = 0  ← 读到最新值
  判断 0 >= 1，不通过
  → 不会超卖
```



**关键：JVM锁包住了整个事务，锁释放时事务已提交，B看到的是最新数据。**

你的理解完全正确。

------

### 三、你的核心问题：为什么不给 SELECT 也加行锁？

**这个问题问到了数据库设计的核心权衡。**

#### 如果 SELECT 也加锁会怎样？

```
BEGIN;
SELECT * FROM product WHERE id = 1 FOR UPDATE;  -- 加排他锁
-- 此时其他事务的 SELECT ... FOR UPDATE 和 UPDATE 都会被阻塞
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```



**这样确实能解决问题**——B的SELECT会被阻塞，等A提交后才能读，读到的是最新值。

#### 但为什么不默认这样做？

| 原因             | 说明                                                         |
| :--------------- | :----------------------------------------------------------- |
| **性能代价太大** | 读操作远多于写操作，如果每次读都加锁，数据库并发能力会急剧下降 |
| **读多写少场景** | 大多数业务是“读多写少”，给读加锁会让系统吞吐量暴跌           |
| **MVCC 的存在**  | InnoDB 用 MVCC 实现“读不加锁也能读到一致快照”，避免读被写阻塞 |
| **锁的粒度问题** | 加锁意味着排队，高并发下会成为瓶颈                           |

**所以数据库的设计选择是：**

- **普通 SELECT**：不加锁，走 MVCC 快照读，性能高
- **需要加锁的读**：显式用 `SELECT ... FOR UPDATE`，由开发者按需使用

**你的直觉是对的：如果给读也加锁，确实能解决这个问题。但代价是性能。所以数据库把选择权交给了开发者——需要强一致时，自己加 `FOR UPDATE`。**

------

### 四、乐观锁依赖行锁吗？

**是的，乐观锁本质上依赖数据库的行锁机制。**

sql

```
UPDATE product SET stock = stock - 1, version = version + 1 
WHERE id = 1 AND version = 0;
```



- 这个 UPDATE 执行时，数据库会给 id=1 这行加行锁
- 如果两个事务同时执行这个 UPDATE，行锁会让它们串行
- 第一个执行成功，version 变成 1
- 第二个执行时，WHERE version=0 不匹配，影响行数为 0

**如果没有行锁，两个 UPDATE 可能同时执行，都认为 version=0 匹配，都成功，版本号机制就失效了。**

所以：

- **行锁**：保证同一时刻只有一个事务能修改这行
- **版本号**：保证修改时数据没有被别人改过
- **两者配合**：行锁是基础，版本号是检测手段

------

### 五、完整总结

| 你的问题                            | 答案                                             |
| :---------------------------------- | :----------------------------------------------- |
| 行锁和事务一起提交吗？              | ✅ 是，事务提交或回滚时才释放                     |
| synchronized 在事务内为什么会超卖？ | ✅ 锁释放时事务没提交，B读到旧值，判断通过后更新  |
| synchronized 在事务外为什么能解决？ | ✅ 锁包住整个事务，B进入时A已提交                 |
| 为什么不给 SELECT 也加行锁？        | ✅ 性能代价太大，读多写少场景会拖垮数据库         |
| 乐观锁依赖行锁吗？                  | ✅ 依赖，行锁保证 UPDATE 串行执行，版本号才能生效 |
| 如果给 SELECT 也加锁能解决吗？      | ✅ 能，但性能差，所以用 `FOR UPDATE` 按需使用     |

**一句话：数据库的设计哲学是“读不加锁，写才加锁”，用 MVCC 保证读的一致性，用行锁保证写的一致性。需要更强一致性时，开发者自己用 `FOR UPDATE` 或乐观锁来补充。**