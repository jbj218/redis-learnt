# 黑马Redis实战篇笔记

## 短信登录功能

==问题1、2解答详见WorkBuddy中的项目疑问==

<h3>1.login中用Session实现直接返回Result.ok(),用Redis实现就要手动定义token返回Result.ok(token)?</h3>

 Session 方案的 JSESSIONID Cookie 由 Tomcat 自动创建和传递，前端完全无感知，所以 `Result.ok()` 就够了,==通过Session传递时自带这个标识，通过session.getAttribute()和session.getAttribute()即可传递==；Redis 方案的 token 是手动生成的普通字符串，没有任何自动传递机制，必须 `Result.ok(token)` 返回给前端，由前端手动存储并携带在 `authorization` 请求头中。

![Session与Redis登录返回值差异](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/Session%E4%B8%8ERedis%E7%99%BB%E5%BD%95%E8%BF%94%E5%9B%9E%E5%80%BC%E5%B7%AE%E5%BC%82.png)

<h3>2.实战篇10中`LoginInterceptor`通过有参构造间接注入`StringRedisTemplate`,而不是直接`@Autowired`注入StringRedisTemplate呢</h3>



**注意**：此时还是没有加双拦截器的版本，加双拦截器后这个逻辑被平移到了RefreshTokenInterceptor

根源在于 **`@Autowired` 只对 Spring 容器管理的 Bean 生效**，而拦截器是 `new` 出来的，不在容器里。

**MvcConfig** 是 `@Configuration` 注解的类，Spring 会把它当作 Bean 管理，所以它内部的 `@Autowired` 能正常工作。拿到 `StringRedisTemplate` 后，再通过构造器参数传递给 `new` 出来的拦截器——这就是**"间接注入"**

**总结**：拦截器在 `MvcConfig` 中通过 `new` 创建，不在 Spring 容器中，所以 `@Autowired` 不生效（运行时为 null）。正确做法是让 `MvcConfig`（`@Configuration`，是 Spring Bean）先 `@Autowired` 拿到 `StringRedisTemplate`，再通过构造器参数传给拦截器。这**是所有 `new` 出来的非 Spring 管理组件获取依赖的标准模式。**

![image-20260724222758664](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260724222758664.png)

<h3>2.1为啥我不能直接在拦截器上加@Component注解让他被spring管理然后再@Autowired将StringRedisTemplate直接注入到这个拦截器上呢？</h3>

**解答：核心是"双实例"问题**

**加 `@Component` 确实可以让 Spring 管理拦截器**。但问题出在：==**如果只加 `@Component` 而不改 `MvcConfig` 里的 `new`==，就会产生==两个实例==。

加了 `@Component` 后会发生：

|                      | 实例 A（Spring 创建） | 实例 B（new 创建）     |
| :------------------- | :-------------------- | :--------------------- |
| **谁创建**           | Spring 容器自动创建   | `new` 关键字创建       |
| **@Autowired**       | ✅ 生效，redis 已注入  | ❌ 不生效，redis = null |
| **被注册为拦截器？** | ❌ 没有                | ✅ 就是这个             |

``addInterceptor()` 接收的是你传进去的那个**实例 B**。你传的是 `new` 出来的，那运行时就用它。Spring 容器里那个 `@Component` 创建的**实例A**呢？它确实存在于容器中，但**从来没有人调用过它的 `preHandle` / `afterCompletion`**，它就是一个闲置的 Bean，占着内存不干活。

**结论:**

**不是"不能"用 `@Component`，而是"没必要"**——构造器方案改动更少、==拦截器保持纯净、不耦合 Spring，是拦截器依赖注入的标准做法==。课程选择构造器方案正是因为它更简洁、更符合拦截器作为 Web 层组件的定位。

![image-20260724224614509](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260724224614509.png)

<h4>也就是说可以通过加@Component的方式来使用@Autowired直接注入,前提是你MvcConfig中的RefreshTokenInterceptor就不能通过new获取而是注入获取了。两种方式不能同时出现</h4>

**@Component方案**==标准用法还是课程中的间接注入==

~~~java
// 第1处：拦截器加注解
@Component
public class RefreshTokenInterceptor implements HandlerInterceptor {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
}

// 第2处：MvcConfig 不能再 new，要注入 bean
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    private RefreshTokenInterceptor refreshTokenInterceptor;  // 注入 Spring 创建的实例

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(refreshTokenInterceptor)  // 用注入的 bean，不用 new
                .addPathPatterns("/**").order(0);
    }
}
~~~

### 实战篇11

<h3>为啥要设置两个拦截器</h3>

==是为了对以登录的用户的巨大优化==

**“查看商品接口”这个动作，不应该要求用户登录（所以不能放在校验拦截器里）；但“查看商品接口”这个动作，必须能帮用户续命（所以必须有一个拦截一切路径的刷新拦截器）。**

| 对比维度           | 单拦截器方案                           | 双拦截器方案（黑马）                               |
| :----------------- | :------------------------------------- | :------------------------------------------------- |
| **看商品 30 分钟** | 无任何拦截器触发，**Token 默默过期**。 | 拦截器 1 触发刷新，**Token 永远有效**。            |
| **30 分后想买**    | 拦截器查不到数据，**被强制踢出**。     | 拦截器 1 刚续过命，拦截器 2 **顺利放行**。         |
| **核心逻辑**       | 校验和刷新混在一起，顾此失彼。         | 剥离出“刷新拦截器（拦截一切）”，独立完成续命工作。 |

场景体现：

<h3>对于已登录用户的场景</h3>

<h3>第一幕：只有一个拦截器时的“30分钟死局” 🚫</h3>

假设只有一个登录校验拦截器，且它只拦截 `/order/**`（下单）这样的敏感接口。

**时间线推演：**

1. **第 0 分钟**：用户登录成功。后端生成 Token（有效期 30 分钟），存入 Redis，并返回给前端。**（此时 Token 生命期开始）**
2. **第 0 - 30 分钟**：用户一直在看商品页面（`/shop/**`）。
   - **关键点来了**：因为只有一个拦截器，且它被配置为**不拦截** `/shop/**`。所以在这 30 分钟内，**用户发出的所有看商品的请求，都没有经过这个拦截器！**
   - **结果**：拦截器根本没有机会去执行“`if (user != null) { 刷新 Token 过期时间 }`”这个动作。Redis 里的 Token 倒计时在**默默地走字**。
3. **第 30 分钟**：Redis 里的 Token 准时过期，被清除了。
4. **第 30 分 01 秒**：用户下单（`/order/add`）。
   - 请求被拦截器拦截。
   - 拦截器拿着前端带来的 Token 去 Redis 查用户信息。
   - **结果：查不到了（因为 Redis 已经删了）！**
   - 拦截器执行 `UserHolder.getUser() == null`，返回 401，强制踢出登录。

**单拦截器的死结：** 你虽然一直“在线”看商品，但因为没触发拦截器，系统就认为你“离线”了，导致关键时刻掉线。

<h3>第二幕：双拦截器如何“破局”？ ⚔️</h3>

现在换成了**拦截器 1（刷新拦截器）** + **拦截器 2（校验拦截器）**。

**时间线推演：**

1. **第 0 分钟**：用户登录成功。
2. **第 0 - 30 分钟**：用户一直在看商品页面（`/shop/**`）。
   - **关键点变了！** 拦截器 1 被配置为**拦截一切路径**（`/*`）。
   - 所以，即便用户在看商品，每一次请求（哪怕是看图片、刷新列表）**都必经拦截器 1**。
   - **拦截器 1 的动作**：`if (token存在 && Redis里有数据) { 执行 expire(key, 30分钟); }`
   - **结果**：因为用户一直在动（滑动、看页），**每隔几秒就有一个请求触发拦截器 1，把 Token 的过期时间重置成“从当前时间再往后推 30 分钟”**。用户一直看，Token 就永远死不了。
3. **第 30 分钟**：用户依然在滑动看商品，拦截器 1 再次刷新 Token 有效期（重置为 30 分钟）。
4. **第 30 分 01 秒**：用户下单（`/order/add`）。
   - 请求先经过拦截器 1：**发现 Token 刚被刷新过，有效！** 拦截器 1 把用户信息存入 ThreadLocal，并放行。
   - 请求接着经过拦截器 2（校验拦截器）：它只拦截 `/order/**`。
   - 拦截器 2 检查 ThreadLocal：发现有数据，`return true`，放行！
   - **结果：下单成功！用户毫无感知，体验极佳。**

<h3>对于最开始没登录的场景</h3>

<h3>未登录用户场景（先逛后买）</h3>

**目标**：看商品是自由的，但想买就得先交“通行证”。

1. **阶段：自由浏览（未登录）**
   - 用户打开网页，没有 Token。
   - 浏览器请求 `/shop/**`（商品详情页）。
   - 后端**拦截器 1**发现没有 Token，**直接放行**；**拦截器 2**不拦截此路径，也**直接放行**。
   - 用户成功看到商品信息，此时 `ThreadLocal` 为空。
2. **阶段：触发购买动作**
   - 用户相中商品，点击“立即购买”按钮。
   - 浏览器携带**空 Token**（或根本没有 Authorization 头），向后端发送 `/order/add` 请求。
3. **阶段：拦截器 1 先行（刷新拦截器）**
   - 它发现请求中没有 Token。
   - 因为无法续命，它**什么也不做**，直接放行，`ThreadLocal` 依然为空。
4. **阶段：拦截器 2 后至（校验拦截器，核心分水岭）**
   - 它拦截了 `/order/add` 请求。
   - **判定**：检查 `ThreadLocal`。此时 `UserHolder.getUser() == null`。
   - **执行动作**：
     - 执行 `response.setStatus(401)`（告知前端：未授权）。==所以校验用户为空返回401不是无缘无故的，是为了返回信息给前端，前端收到反馈以进行下一步动作==
     - 执行 `return false`（停止请求转发，避免进入 Controller）。
   - **后端执行完毕，响应用户**。
5. **阶段：前端接管（跳转登录）**
   - 前端（Vue/Axios）的拦截器捕捉到了 HTTP 状态码 **401**。
   - **前端动作**：自动触发路由跳转，把页面**强制切换回登录页**。
   - **结果**：用户看到登录界面，需要输入手机号和验证码完成登录。

> ==为什么不在登录时就保存到userholder中而是在拦截器保存用户？==

因为**登录只发生一次，而“获取当前用户”是每个请求都要做的事**——这两个场景根本不是一回事。

==<h4>关键前提：HTTP 是无状态的</h4>==

登录那一刻，你确实知道用户是谁，也确实可以把用户存进 ThreadLocal。但问题是：**登录请求结束后，这次存储就随着请求结束了**。

之后的每一个请求（查店铺、下单、点赞……）都是**全新的请求**，服务器不会记得“他登录过”。Tomcat 用线程池复用线程，处理下单请求的线程很可能不是登录那个线程——而 ThreadLocal 是**线程私有**的，登录线程里存的东西，别的线程根本拿不到。

plaintext

```
登录请求 → 线程A 的 ThreadLocal 存了 user  → 请求结束，这个存储就没意义了
下单请求 → 分到线程B → 线程B 的 ThreadLocal 是空的 → 拿不到用户 ❌
```

所以每个请求都必须**重新确认“我是谁”**。唯一能证明身份的就是请求头里的 token，于是流程只能是：每个请求进来 → 拿 token → 去 Redis 查用户 → 存进**当前线程**的 ThreadLocal → 业务代码随时取用。这个“每个请求都要做”的工作，就是拦截器的活。

<h4>为什么放拦截器而不是别的地方</h4>

| 方案                                                 | 问题                                             |
| :--------------------------------------------------- | :----------------------------------------------- |
| 登录时存                                             | 只对登录那一次请求有效，后续请求拿不到（无状态） |
| 存静态变量共享                                       | 所有线程共用一个用户，并发下用户串号             |
| 每个 Controller/Service 里自己写“取token→查Redis→存” | 每个接口都抄一遍，重复代码泛滥                   |
| **拦截器统一做**                                     | ✅ 所有请求自动经过，写一次全生效                 |

拦截器是 Spring MVC 提供的“**所有请求的统一入口**"——不管请求打到哪个 Controller，都先过拦截器。在这里做“取用户→存 ThreadLocal”，业务代码里就只需要一行 `UserHolder.getUser()`，干净且不会漏。

<h4>打个比方</h4>

- **登录** = 第一次办会员卡（办完就走了，卡不在店里）
- **token** = 你每次进店出示的会员卡
- **拦截器** = 门口的刷卡机——每个人进门都刷一次，确认身份后在本子上登记（ThreadLocal），店内所有服务员（Controller/Service）随时翻本子就知道当前顾客是谁
- **afterCompletion 清理** = 顾客出门后把本子撕掉，防止下一位顾客被当成上一位

<h4>一句话</h4>

登录只解决“发 token + 存 Redis”这一件事；之后**每个请求都是独立的、无状态的**，必须靠拦截器在每个请求开头“拿 token → 查 Redis → 存 ThreadLocal”重新确认身份。存登录时只救得了那一个请求，存拦截器才能让所有接口都拿到当前用户。

### 关键误区澄清

> 1、也就是说ThreadLocal的本质作用是方便业务调用，而不是所有线程共享。真正验证用户登录的永远是token？
> 2、之前一直以为，发送的请求是叠加式的，我进入一个页面然后再这个页面点进另一个页面是基于当下页面来寻路的？实际是每个请求都是独立的是吗？如果需要验证登录状态，每个独立的请求都要自己验证一次？

两个理解都对，确认并补一点细节。

#### 问题1：对，准确表述是

**token 是身份凭证（唯一真相），ThreadLocal 只是“本次请求内的临时提货单”。**

- 验证登录 = 拿 token 去 Redis 查，查得到就是已登录——这是每个请求都要做的真验证
- ThreadLocal 的作用是：拦截器验证完一次之后，把结果（UserDTO）挂在当前线程上，**让后面的业务代码不用重复去查**。它只在“这一次请求”内有效，线程之间互相隔离，不是共享

所以分工是：

| 角色                     | 职责                 | 生命周期                       |
| :----------------------- | :------------------- | :----------------------------- |
| token                    | 证明"我是谁"         | 跨请求，长期有效（存在 Redis） |
| Redis 里的用户信息       | token 对应的真实数据 | 跨请求，长期有效               |
| ThreadLocal 里的 UserDTO | 一次请求内的便利缓存 | 一次请求，用完就删             |

没有 ThreadLocal 也能活——每个 Service 里都自己拿 token 查 Redis 就行，只是要写几百遍重复代码。ThreadLocal 纯粹是为了方便。

#### 问题2：对，每个请求都是完全独立的

==HTTP 协议天生就是**无状态**的==。你感觉“从页面 A 点进页面 B 是接着 A 走的”，那只是**浏览器**的记忆（URL、历史记录、前端路由），服务器对此毫不知情。

服务器眼里，每个请求都是一封**自包含的信**：

plaintext

```
请求 = 方法 + URL + 请求头 + 请求体

POST /orders  请求头: authorization: abc123
```

服务器读这封信时，**只知道信上写了什么**，不知道你上一封请求是什么、从哪个页面点过来的。处理完就忘，下一个请求来了又是从零开始。

所以：

**是的，每个独立的请求都要自己验证一次登录。** 验证的方式就是每个请求都带上 token（浏览器/前端会自动带上，你不用手动操作），拦截器每次都查一遍 Redis。

#### 补一个容易困惑的点

“那登录接口岂不是每秒被调用无数次？”不是。**验证登录 ≠ 执行登录**：

- **登录**（发验证码、核对验证码、发 token）：只做一次，产出 token
- **验证**（拿 token 查 Redis 有没有这个用户）：每个请求做一次，就是一次 Redis 的 `get`，非常轻

类比：**登录是办会员卡（一次），验证是每次进店刷卡（每次，但刷卡只要一秒）**。HTTP 不帮任何一方“记住”状态，所谓“登录态”完全靠客户端每次主动出示 token + 服务端每次主动查验撑起来的。

一句话：**HTTP 每个请求都是独立自包含的，“我登录过”这个状态服务器从不保存，靠的是每个请求都带着 token、每个请求都被拦截器独立验证一次；ThreadLocal 只是把验证结果在单次请求内传递下去，方便业务取用。**

## 商户查询缓存

### 实战篇10 互斥锁

~~~java
    //获取锁
    private boolean tryLock(String key){
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }
~~~

> 1.获取锁逻辑中直接返回flag不行吗?

<h3>核心原因：防止 `flag` 为 `null`</h3>

`stringRedisTemplate.opsForValue().setIfAbsent(...)` 的返回值类型是 `Boolean`（包装类，不是基本类型 `boolean`）。

- **正常情况下**：Redis 执行成功，返回 `true`（获取锁成功）或 `false`（获取锁失败）。
- **异常情况下**：如果 Redis 网络超时、连接断开，或者执行过程中抛出异常，**这个方法可能返回 `null`**。

如果你直接 `return flag;`，当发生异常时，上层调用者收到 `null`，可能会引发以下问题：

- **空指针异常**：如果上层代码有 `if (flag)` 这种自动拆箱操作，就会直接抛出 `NullPointerException`。
- **逻辑混乱**：上层可能把 `null` 错误地当作“没拿到锁”处理，导致业务逻辑出错。

而 `BooleanUtil.isTrue(flag)`（来自 Hutool 工具包）的源码逻辑是：

```java
public static boolean isTrue(Boolean bool) {
    return bool != null && bool;
}
```

**这样即使发生异常，也只会返回 `false`（没拿到锁），保证了方法的健壮性。**

>2.为什么要给锁设置有效期?基础期间看2.1即可

**一句话概括：设置过期时间是为了防止“死锁”**，也就是防止锁永远不被释放。

<h3>2.1 不设时间会发生什么？（灾难现场）</h3>

假设你写的是这样的代码（不设过期时间）：

```java
// 1. 获取锁
Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent("lock:order", "1");
if (flag) {
    // 2. 执行业务逻辑（比如查询数据库、扣库存）—— 耗时 5 秒
    doBusiness(); 
    
    // 3. 释放锁
    stringRedisTemplate.delete("lock:order");
}
```

**如果第 2 步 `doBusiness()` 执行过程中发生异常，会发生什么？**

- **情况一**：代码抛出异常，`delete` 释放锁的代码**根本不会执行**。
- **情况二**：执行 `doBusiness()` 的线程突然宕机、重启，或者网络断开了。

**结果：** Redis 里的 `lock:order` 这个 Key 永远存在，**永远不会被删除**。后面的所有线程尝试 `setIfAbsent` 都会失败，整个系统就**死锁**了（所有请求都被卡住，无法处理）。

<h3>2.2 但是，设了时间就一定能保证安全吗？</h3>

**设置过期时间确实解决了“永远不释放”的问题，但引出了一个新的致命问题——**“锁误删”**。**

**经典陷阱示例：**

```java
// 线程 A 拿到锁，设置 10 秒过期
Boolean flag = redis.opsForValue().setIfAbsent("lock", "A", 10, TimeUnit.SECONDS);
// 业务执行了 15 秒（超过 10 秒）
doBusiness();  // 耗时 15 秒

// 此时锁已经自动过期了，线程 B 可以拿到锁（Key 被删了）

// 线程 A 执行到这里，执行删除
redis.delete("lock");  // ❌ 这里删的是线程 B 刚加的锁！
```



**结果：** 线程 A 把线程 B 刚加的锁给删了，导致线程 B 的业务还没执行完，锁就没了，两个线程同时操作共享资源，数据就乱了。

<h3>2.3 那到底怎么解决这两个问题？</h3>

正因为如此，在生产环境中，**“设置过期时间”只是第一步**，必须配合以下方案才能成为一把“正经”的分布式锁：

<h4>方案一：设置过期时间 + 锁的归属判断（原子删除）</h4>

在释放锁时，必须**判断当前锁是不是自己加的**，并且**判断和删除必须是原子的**（用 Lua 脚本）：

```java
// 加锁时，value 设为唯一标识（如 UUID）
String uuid = UUID.randomUUID().toString();
Boolean flag = redis.opsForValue().setIfAbsent("lock", uuid, 10, TimeUnit.SECONDS);

// 释放锁时，用 Lua 脚本保证原子性（判断 UUID 是否匹配，匹配才删除）
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
redis.execute(new DefaultRedisScript<>(script, Long.class), 
              Collections.singletonList("lock"), uuid);
```

<h4>方案二：使用 Redisson（自动续期，看门狗机制）</h4>

**最优雅的做法**是直接用 Redisson 框架，它会自动帮你做两件事：

1. **给锁设置 30 秒默认过期时间**
2. **启动一个“看门狗”线程**：每过 10 秒检查一次，如果业务还没执行完，就**自动帮你在 Redis 里把过期时间重置为 30 秒**，防止业务还没跑完锁就先过期了。

```java
RLock lock = redissonClient.getLock("lock");
lock.lock(10, TimeUnit.SECONDS);  // 如果业务超过 10 秒，看门狗会自动续期
try {
    doBusiness();
} finally {
    lock.unlock();
}
```



> 3.为什么做缓存击穿-互斥锁 逻辑时，成功获取锁后，还要再检查一次redis缓存是否存在。

关键在于：**第一次查缓存和拿到锁之间有时间差，别的线程可能已经把缓存重建好了。**

想象这个时间线：

```
Thread A                          Thread B
────────                          ────────
① 查 Redis → null
② 抢锁 → 成功 ✓
③ 查数据库...（很慢）
                                  ① 查 Redis → null
                                  ② 抢锁 → 失败，休眠 50ms
                                  ② 抢锁 → 失败，休眠 50ms
                                  ② 抢锁 → 失败，休眠 50ms
④ 写入 Redis ✓
⑤ 释放锁 ✓
                                  ② 抢锁 → 成功 ✓
                                  ← 此时 Redis 里已经有数据了！
                                  ③ 如果不二次检查 → 白查一次数据库
                                  ③ 二次检查 → 发现缓存已存在 → 直接返回 ✓
```

Thread B 在**第一次查 Redis 时**（步骤①）缓存还是空的，但在**抢到锁时**（步骤②成功）Thread A 早已把缓存写好了。如果不二次检查，B 会做一次完全不必要的数据库查询。

这就是经典的 **Double-Checked Locking（双重检查锁定）** 模式——第一次检查在锁外，第二次检查在锁内。

### 实战篇11 逻辑过期

> ==1、获取具体shop对象的逻辑中为啥不直接强转(涉及对强转理解的误区)==

~~~Java
Shop shop = (Shop)redisData.getData();
~~~

反而用

~~~java
Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
~~~

**问题的关键在于 JSONUtil 在处理嵌套对象类型的数据时，由于这个嵌套数据由StringRedisTemplate查出来的不自带数据类型。所以在`RedisData redisData = JSONUtil.toBean(json, RedisData.class);`的过程， JSONUtil 只能把嵌套的对象还原成一个通用的 `JSONObject`（本质是个 Map）。**

**`JSONObject` 和 `Shop` 是两个完全不同的类，没有继承关系，强转会直接 `ClassCastException`。**

只能使用`JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class)`做对象映射，把 Map 里的字段按字段名反射拷贝到 Shop 对象里。



tip:首先用Shop直接强转会报错类型不匹配

~~~bash
java.lang.ClassCastException: class cn.hutool.json.JSONObject cannot be cast to class com.hmdp.entity.Shop (cn.hutool.json.JSONObject and com.hmdp.entity.Shop are in unnamed module of loader 'app')
~~~

前置知识1：

对于==StringRedisTemplate==这种获取数据的方法来说，==反序列化出来的数据是不会记录存入数据原本的数据类型==的，只能通过序列化工具手动将数据映射到指定类型的对象中。

前置知识2：

JSONUtil.toBean（）的原理是将对应的数据映射到指定的数据接口中，和强转有本质区别。

**解答:**

```java
public class RedisData {
    private LocalDateTime expireTime;
    private Object data;    // ← Object 类型
}
```

`data` 声明为 `Object`。序列化整个 `RedisData` 对象为 JSON 存入 Redis 后，再反序列化回来时，JSON 里没有"这个 Object 原本是 Shop"的类型信息，==所以 JSONUtil 只能把它还原成一个通用的 `JSONObject`（本质是个 Map）。==

```java
// Redis 里的 JSON：
{
  "expireTime": "2026-07-31T10:00:00",
  "data": {              // ← 这是个嵌套对象
    "id": 1,
    "name": "百味鲜面"
  }
}
// 反序列化后：
redisData.getData() → JSONObject（不是 Shop）
```

`JSONObject` 和 `Shop` 是两个完全不同的类，没有继承关系，强转会直接 **`ClassCastException`**。所以必须用 `JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class)` 做一次"映射转换"——把 Map 里的字段按字段名反射拷贝到 Shop 对象里。

> 1.1(JSONObject) redisData.getData()为啥这里还要强转一下呢？获取的本身不就是JSONObject类型吗

因为**声明类型**和**运行时类型**是两回事。

```java
// RedisData 中 data 的声明
private Object data;  // ← 编译时类型是 Object

// 方法签名
public Object getData() { return data; }  // ← 返回值声明为 Object
```

`JSONUtil.toBean()` 有多个重载版本：

```java
JSONUtil.toBean(Object source, Class<T> clazz)     // 传 Object → 内部再判断类型，多一层开销
JSONUtil.toBean(JSONObject source, Class<T> clazz)  // 传 JSONObject → 直接映射，更高效
```

你不强转的话，`redisData.getData()` 在编译器眼里就是 `Object`，会走到第一个重载——虽然也能工作，但多了一层反射判断。

加了 `(JSONObject)` 强转后，告诉编译器："别看它是 `Object`，我保证它是 `JSONObject`"，于是精确匹配到第二个重载，路径更直接。

**简单说：不是给 JVM 看的（运行时本来就是 JSONObject），是给编译器和重载方法选择看的。** 不加也能跑，但加上了方法调用更精准，代码意图也更明确。

> 1.2 反正你返回的就是Object，我将Object转成Shop也不犯毛病吧？

==强转的误区：强转不是你是个Object就能直接转成任意的数据类型了==

==强转的要求==

**前提 1：存在直接的父子类或接口实现关系（语法要求）**

**你只能把“父类引用”强转为“子类”，或者把“接口”强转为“实现类”。**
如果你尝试把两个完全不沾边的类（比如 `String` 和 `ArrayList`）进行强转，编译器直接就会报错，连编译都通不过。

**==前提 2：该对象的“运行时实际类型”必须是目标类型或其子类（核心要求！）==**

这是**最容易踩坑**的地方。编译器只看类型声明，但 **JVM 运行时看的是对象在堆内存里真正的模样**。

Java 的强转**不是"换个声明类型就行"**，JVM 在运行时会真的去检查这个对象的**实际类型**是不是 `Shop`。

```
内存中的对象 ──→ 实际类型：JSONObject

(Object) obj           → ✅ 没问题，声明类型变 Object
                      
(Shop) obj             → ❌ JVM 一查：你这个对象不是 Shop 也不是 Shop 的子类
                          ClassCastException！
```

强转不是魔法，不能把一个 `JSONObject` 凭空变成 `Shop`。`JSONUtil.toBean()` 之所以能做这件事，是因为它在**内部遍历字段、反射赋值、重新构造了一个全新的 `Shop` 对象**——这是"转换"，不是"强转"。

> 1.3回忆Redis入门篇中6.2 数据序列化器篇章，对StringRedisTemplate有了更深的理解

**对于StringRedisTemplate这种方式来说取出来的类型都是不带数据本身的java类型的，只能通过序列化工具将数据映射到指定类型的对象中。而如果你自定义RedisTemplate的序列化方式，就可以自动反序列出指定对象，缺点是内存开销更大**

| StringRedisTemplate | RedisTemplate（JDK 序列化）     | RedisTemplate（JSON 序列化）   |                            |
| :------------------ | :------------------------------ | :----------------------------- | -------------------------- |
| **存什么**          | `String`                        | JDK 序列化字节流               | JSON 字符串                |
| **Redis 里能看到**  | ✅ 明文 JSON                     | ❌ 二进制乱码                   | ✅ 明文 JSON                |
| **反序列化**        | 手动 `JSONUtil.toBean()`        | 自动还原                       | 自动还原                   |
| **类型信息**        | ✅ 无，所有相关代码清清楚楚      | ❌ 字节流里埋了类名，换包名就炸 | ❌ 依赖 `@class` 或类型配置 |
| **内存开销**        | 最小                            | **最大**（字节流膨胀）         | 中等                       |
| **可调试性**        | 直接 `redis-cli get key` 就能看 | 看不懂                         | 能看懂                     |

<h4>什么时候适合自定义 RedisTemplate 序列化</h4>

当你的==对象结构稳定==、==不需要在 Redis 里手动查看==、且希望==省掉每次 `toBean` 的重复代码==时，用 Jackson 序列化的 `RedisTemplate` 会更方便——但代价是你刚刚说的：多一些序列化元数据开销，以及 Redis 里的数据绑定死了 Java 类型。

**总结：`StringRedisTemplate` = 手动但可控；`RedisTemplate` 自动序列化 = 方便但耦合。** 课程选前者是因为教学场景下，开发者应该清楚每一步在做什么。

> 2、获取锁成功并重建缓存的逻辑中，为啥要开辟一个独立线程来执行，直接执行不行吗？(2,3本质同一个问题)

=="重建缓存的事交给后台线程慢慢干" 是解决2、3问题的本质==

**可以直接执行，但就丧失了"逻辑过期"的意义。**

逻辑过期相比互斥锁的核心优势就是：**不让用户等**。

|                | 互斥锁方案                  | 逻辑过期（直接执行）        | 逻辑过期（独立线程）             |
| :------------- | :-------------------------- | :-------------------------- | :------------------------------- |
| 拿到过期数据后 | 等锁 → 查库 → 写缓存 → 返回 | 等锁 → 查库 → 写缓存 → 返回 | 等锁 → 立刻返回旧数据 → 后台重建 |
| 用户等待时间   | 查数据库的全程              | 查数据库的全程              | 几乎为 0                         |

如果直接执行，重建缓存的完整链路（抢锁 → 查数据库 → 写 Redis）全在请求线程里，用户一样要等——那就和互斥锁方案没区别了。

开独立线程是"逻辑过期"的设计精髓：**当前用户拿到旧数据马上走，==重建缓存的事交给后台线程慢慢干==，下个用户就能拿到新数据了。**

> 3、重建缓存成功后，是不是应该再返回最新的shop啊？这里还是返回旧的shop吗？

**不应该，也做不到。**

看代码结构就知道了：

```java
// 主线程：已经拿着旧 shop 准备返回了
CACHE_REBUILD_EXECUTOR.submit(() -> {
    this.saveShop2Redis(id, 20L);  // 后台线程在重建...
});
// 171 行：提交任务后，主线程继续往下走

// 181 行：返回旧 shop
return shop;
```

submit 是**异步**的，`()` -> `{...}` 里的代码在另一个线程执行，主线程 `return shop` 时，重建可能还没开始。

这是一个"舍小保大"的权衡：**当前这个用户拿到的是过期数据（边界情况，概率低），但所有后续用户拿到的都是新鲜数据。** 对热点数据的整体体验来说，这是更优解。

## 优惠卷秒杀

### 实战篇02

> RedisIdWorker中 return timestamp << COUNT_BITS | count 为啥能够实现拼接呢？或运算不就是或吗？他还能加吗？

~~~java
return timestamp << COUNT_BITS | count;
~~~

这个问题的关键在于：你脑子里的"或"是**逻辑或**（`||`，对 true/false 运算），而这里的 `|` 是**按位或**——对每一位独立运算，它们不是一回事。

<h4>按位或的运算规则</h4>

`|` 会对两个数的**每一位**分别做"或"运算，规则就四条：

| 左位 | 右位 | 结果 |
| :--- | :--- | :--- |
| 0    | 0    | 0    |
| 0    | 1    | 1    |
| 1    | 0    | 1    |
| 1    | 1    | 1    |

关键在第二条和第三条：**`0 | x = x`**——0 跟任何位做"或"，结果就是那个位本身。这就是"拼接"的全部秘密。

<h4>为什么能拼？看图里的三行</h4>

**第一行 `timestamp << 32`**：timestamp 左移 32 位后，原本在低位的值被推到了高 32 位，而低 32 位被空出来的位置全填了 0。所以它的结构是 `[timestamp值 | 32个0]`。

**第二行 `count`**：count 是 Redis 自增得到的序列号，它只占低 32 位（前提是没超过 2³² ≈ 42.9 亿），高 32 位天然全是 0。结构是 `[32个0 | count值]`。

**两行做 `|`**：按位或逐位计算——

- 高 32 位：timestamp 的值 `|` 0 = timestamp 的值（`x | 0 = x`）
- 低 32 位：0 `|` count 的值 = count 的值（`0 | x = x`）

结果就是 `[timestamp值 | count值]`——两段比特被拼在了一起。

<h4>它确实不是"加"</h4>

你的直觉没错——`|` 不是加法。它能"拼"的前提是**两段比特不重叠**：timestamp 那部分 count 全是 0，count 那部分 timestamp 全是 0。正因为不重叠，每一处要么是 `x | 0`、要么是 `0 | x`，结果就是"各取各的"。

如果两段重叠了（都有 1），`1 | 1 = 1`，那就不是拼接而是"合并覆盖"了。所以这个方案有一个**隐含前提**：count 不能超过 2³² - 1（约 42.9 亿），否则 count 会溢出到高 32 位，和 timestamp 的比特撞上，拼出来的 ID 就错了。Redis 的 `increment` 在单 key 单天内不可能自增 42 亿次，所以这个前提是成立的。

<h4>一个具体例子</h4>

假设 timestamp = 5（二进制 `101`），count = 3（二进制 `11`），COUNT_BITS = 32：

```
timestamp << 32:  0000...00000101 00000000000000000000000000000000
                                    ↑ 低32位全0
count:            0000...00000000 00000000000000000000000000000011
                  ↑ 高32位全0
                  ─────────────── | ──────────────────────────────
结果:             0000...00000101 00000000000000000000000000000011
                  ↑ 高32位=timestamp   ↑ 低32位=count
```

高 32 位拿到了 timestamp，低 32 位拿到了 count，两段比特完美拼接成一个 64 位整数。不是加法，是利用 `0 | x = x` 让两段不重叠的比特各归各位。

![image-20260804205850469](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260804205850469.png)

> 位运算比较的是Long类型对应的2进制吗？所有基本类型比较的都是底层的二进制？

<h4>位运算操作的就是底层二进制</h4>

`timestamp` 和 `count` 虽然是 `long` 类型，但在内存里它们就是 **64 个 0/1 比特**。`<<`、`|`、`&` 这些位运算符根本不关心这个数"代表"几——它们直接对这 64 个比特逐位操作。

拿 `5 | 3` 举例，5 的二进制是 `101`，3 的二进制是 `011`，按位或就是对每一位分别做"或"：

```
  1 0 1   (= 5)
| 0 1 1   (= 3)
───────
  1 1 1   (= 7)
```

运算前是"五"和"三"，运算时只看二进制位，运算完再重新解释成"七"。位运算从头到尾都在操作二进制，"数值"只是人给这些二进制起的名字。

但要注意，Java 里==**只有整数类型**==能做位运算，而且每种类型的二进制宽度不同：

| 类型      | 二进制宽度             | 能做位运算 |
| :-------- | :--------------------- | :--------- |
| `byte`    | 8 位                   | 能         |
| `short`   | 16 位                  | 能         |
| `int`     | 32 位                  | 能         |
| `long`    | 64 位                  | 能         |
| `char`    | 16 位（存 Unicode 码） | 能         |
| `float`   | 32 位                  | **不能**   |
| `double`  | 64 位                  | **不能**   |
| `boolean` | 1 位                   | **不能**   |

`float` 和 `double` 底层也是二进制，但它们的 64 个比特是用 **IEEE 754 标准**编码的（符号位 + 指数位 + 尾数位），直接拿这些比特做 `|`、`&` 没有任何意义——你不会想把指数位和尾数位"或"在一起。所以 Java 直接禁止了浮点数做位运算。`boolean` 只有 true/false 两个值，也没有按位运算的必要。

<h4>一句话总结</h4>

计算机内存里一切数据本质上都是二进制比特，位运算就是绕过"数值含义"直接操作这些比特。`long` 是 64 位二进制，`|` 就是对这 64 位逐位做"或"。所有整数类型都是这个逻辑，只是宽度不同；==浮点数虽然也是二进制，但编码方式特殊，直接按位操作没意义，所以 Java 不允许。==

> 测试类中testIdWorker中的latch.countDown();放在  es.submit(task);后面效果一样吗？

<h4>区别对比</h4>

| 位置                            | 执行时机                     | 效果                                   |
| :------------------------------ | :--------------------------- | :------------------------------------- |
| ✅ **在 `task` 内部**            | 每个线程任务**执行完成**后   | 正确：等所有任务真正做完才统计时间     |
| ❌ **在 `es.submit(task)` 后面** | 任务**刚提交**到线程池就执行 | 错误：线程池可能还没执行，就认为完成了 |

<h4>为什么放后面会出错？</h4>

假设你把代码改成这样（错误）：

```java
for (int i = 0; i < 300; i++) {
    es.submit(task);
    latch.countDown();  // ❌ 错误位置
}
```

流程变成：

1. 提交一个任务到线程池（可能还在队列里等待）
2. **立即**执行 `latch.countDown()`（计数器减 1）
3. 这 300 次循环瞬间执行完，`latch` 瞬间变成 0
4. `latch.await()` 立即通过，**此时任务可能一个都没真正运行**
5. 统计的时间只是提交任务的时间，不是执行时间

### 实战篇04

> 为啥每个秒杀卷不单独代表一行，而是以stock库存的形式代表一堆秒杀卷

这是个很好的数据库设计问题。核心答案是一个词：**可互换性**。秒杀券之间是"一模一样、可互换"的，你抢到哪张都一样，所以不需要给每张券单独的身份。但只说这个还不够，更关键的差异在**并发抢券时的处理方式**上。先用图对比两种方案。

![image-20260806092450242](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260806092450242.png)

<h4>原因一：券是"可互换的"，没有独立身份</h4>

你买的"100元代金券"，第1张和第2张有区别吗？没有。标题一样、面额一样、规则一样、生效时间一样——它们是**可互换的**（fungible），你抢到哪张都无所谓。

既然每张券之间没有任何区别，给它们各分一行就毫无意义：100行数据内容一模一样，除了 id 不同，所有字段值完全相同。这叫**冗余存储**——存了一堆"长得一模一样"的行，却没有传递任何额外信息。用 `stock = 100` 一个数字就能表达"有100张"，何必用100行来表达同一件事？

<h4>原因二（关键）：并发扣减一条SQL搞定</h4>

这才是秒杀场景最硬的理由。看你项目第172-175行的真实代码：

```java
boolean success = seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId).gt("stock", 0)
        .update();
```

等价于一条 SQL：

```sql
UPDATE tb_seckill_voucher SET stock = stock - 1 WHERE voucher_id = ? AND stock > 0
```

3个用户同时抢：**三个请求都打这一行**，数据库靠行锁自动排队串行执行。`stock > 0` 这个条件保证库存归零后直接不更新（返回0行受影响=抢失败）。**一条SQL，原子完成"判断+扣减"，不需要查了再改。**

如果逐行存储呢？你得**两步**：

1. `SELECT id FROM ... WHERE status = '可用' LIMIT 1` — 找一张可用的
2. `UPDATE ... SET status = '已用' WHERE id = ?` — 标记为已用

问题来了：U1 和 U2 的 `SELECT` 可能**同时选中同一行**（比如都选中了 id=5），然后都去 `UPDATE` 它。只有一个 `UPDATE` 成功（影响1行），另一个发现行已经被占了（影响0行），就得**重试**——再 `SELECT` 找下一张。秒杀并发几万人，这种冲突会频繁发生，重试逻辑复杂、性能差。

<h4>对比表</h4>

| 维度            | stock字段                           | 逐行存储                               |
| :-------------- | :---------------------------------- | :------------------------------------- |
| 100张券占多少行 | 1行                                 | 100行                                  |
| 抢券SQL         | 一条 `UPDATE stock-1 WHERE stock>0` | `SELECT可用行` + `UPDATE标记已用` 两步 |
| 并发冲突        | 行锁自动串行，无冲突                | 多人可能选中同一行，需重试             |
| 适合场景        | 可互换的券（秒杀券）                | 有独立身份的券（兑换码、座位票）       |

<h4>总结：</h4>

秒杀券用 `stock` 而不用逐行，是因为它们**可互换**（没独立身份，逐行是冗余），而且秒杀是**高并发扣减**场景——一条 `UPDATE stock-1 WHERE stock>0` 原子搞定，比逐行的"先查再改+冲突重试"简单可靠得多。只有每张券有唯一身份（兑换码、座位号）时，才值得逐行存储。



### 实战篇07（涉及锁和事务传播，很重要）

> 小问题：这里为啥不写count>1来判断用户已近购买过一次呢？

~~~java
if (count > 0) {
            // 用户已经购买过了
            return Result.fail("用户已经购买过一次！");
        }
~~~

一句话：**`count > 0` 是"只要有一条就拦后续"，在第二单发生之前就拦住；`count > 1` 是"等有了两条才拦"，第二单已经漏进去了。** 检查的目的就是在第二单写入之前拦住它，不是等第二单写完之后才发现"哦多了"。



> 为了更细的粒度锁userId时：为啥要这样synchronized(userId.toString().intern()){...}

1. 为什么不用synchronized(userId)?

   `userId` 是 `Long`（包装类）。`synchronized` 锁的是**对象地址**，不是值。

   问题在于：

   ==①`Long` 是包装类，当值超出 [-128, 127] 缓存范围时（用户 ID 通常都是大数），**每次 `getId()` 返回的都是一个全新的 `Long` 对象**。==两个线程拿着值都是 1010 的 `Long`，但它们是**两个不同的对象**（`Long@0x1A` 和 `Long@0x2B`），`synchronized` 各锁各的，互斥完全失效。

   ==②即使值恰好在缓存范围内，也不该锁包装类==——别的代码可能也锁了同一个缓存对象，造成意外的锁竞争。

   UserDTO不同——>Long 类型的userId不同

   <h4>为什么不同对象的根源</h4>

   ①ThreadLocal 本身的机制：==每个线程有独立的存储槽==

   ThreadLocal 内部维护了一个 `ThreadLocalMap`，挂在每个 `Thread` 对象上。`tl.set(userDTO)` 实际上是往**当前线程自己的 map** 里塞了一个条目。所以 `tl.get()` 在不同线程里返回的天然是不同东西——这是 ThreadLocal 这个类提供保证的，不需要你写任何额外代码。

   ②代码设计的机制：==每次 `new UserDTO()` 创建新对象==

   看 `RefreshTokenInterceptor.java` 第40行：

   ```java
   UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
   ```

   这个 `new UserDTO()` 是你代码里写的，不是 ThreadLocal 干的。它决定了"每次创建一个新对象"。

2. 为什么不用synchronized(userId.toString)?

   `toString()` 确实把 `Long` 变成了 `String`，但问题没解决：==**`Long.toString()` 每次调用都会在堆里 new 一个新的 String 对象**==。两个线程都调用 `toString()`，拿到的是内容相同（"1010"）但地址不同（`String@0x3C` 和 `String@0x4D`）的两个 String。`synchronized` 还是各锁各的

3. intern()有啥作用？

   `intern()` 是 `String` 类的方法，它会去 ==**JVM 字符串常量池**==里查：

   - 常量池里已经有 "1010" → 直接返回池子里那个对象的引用
   - 常量池里没有 → 把当前 String 放进池子，再返回它的引用

   所以 `userId.toString().intern()` 保证了：**不管调用多少次，只要值相同，返回的都是常量池里同一个 String 对象**。两个线程拿到的都是 `String@POOL`（同一地址），`synchronized` 终于锁住了同一把锁，互斥成功。

<h4>一句话总结</h4>

| 问题                                            | 答案                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| Q1 为什么不用 `synchronized(userId)`            | `Long` 是包装类，每次 `getId()` 返回新对象，两个线程锁的是不同对象 |
| Q2 为什么不用 `synchronized(userId.toString())` | `toString()` 每次 new 一个 String，还是不同对象              |
| Q3 `intern()` 啥作用                            | 从字符串常量池取同一引用，保证相同值 → 同一对象 → 同一把锁   |



> 为啥事务注解放在private上会报错(涉及事务注解的机制，下面的自调用问题也会提到)

![image-20260805203836752](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260805203836752.png)

`@Transactional` 不是你自己执行的——**Spring 用 AOP 代理帮你执行的**。流程是这样的：

1. Spring 启动时，用 CGLIB 给你的类生成一个**代理子类**（图上半部分）
2. 代理子类**重写你的 public 方法**，在方法前后织入事务逻辑：`开启事务 → super.调用你的原始方法 → 提交/回滚事务`
3. 别人注入的 `VoucherOrderServiceImpl` 其实不是你写的那个对象，而是**代理子类的实例**

关键问题在"重写"这一步。**Java 语法规定：`private` 方法不被子类继承，子类无法 `@Override` 它**。代理子类压根看不到你的 `private` 方法，更别提重写它来加事务了。所以 `@Transactional` 标在 `private` 上：

- 代理无法拦截这个方法 → 事务逻辑织不进去 → **`@Transactional` 形同虚设**
- 在 Spring 5.x（你项目的版本）中会被静默忽略（事务不生效但不报错）；在更新版本中会直接报错

改成 `public` 后，代理子类就能重写它，事务逻辑才能织入。

==**补充一个坑**==：就算改成了 `public`，还有**自调用问题**。在第260行 `seckillVoucher` 里调 `createVoucherOrder` 是 `this.createVoucherOrder()`——`this` 指向的是原始对象，不是代理对象，**绕过了代理，事务同样不生效**。这就是为什么这段代码最终被注释掉了，改用了 Redisson 分布式锁（第147行那个 `createVoucherOrder`）。



> 下个问题前先简单理解一下Mysql中事务的机制，事务的快照机制

==关键点：数据库有个规则，**一个事务改的数据，在它提交之前，其他事务查的时候看不到这次改动。** 查到的是改动之前的旧数据。==

MySQL InnoDB 默认隔离级别是 **REPEATABLE READ（可重复读）**，它的核心机制叫 **MVCC（多版本并发控制）**，每个事务都会有一份"快照"：

plaintext

```
时间线 ─────────────────────────────────────────────────────►

       事务B开始           A提交事务              B查count
         │                  │                    │
         ▼                  ▼                    ▼
   ┌────────────────┐    ┌────────┐    ┌────────────────┐
   │ B 拍快照       │    │ 数据库 │    │ B 查 count     │
   │ (看到旧版本)   │    │ 已更新 │    │ 走快照，仍是0  │
   └────────────────┘    └────────┘    └────────────────┘
```

**关键规则**：

1. **快照在事务第一次 SELECT 时建立**（更精确：第一次"快照读"时），之后这个事务内所有的**普通 SELECT** 都读这个快照版本
2. **A 的提交对 B 的快照不可见**——A 改了数据，但 B 的事务从一开始就读旧版本，A 提交后 B 的快照不会被刷新
3. 这就是 REPEATABLE READ 的"可重复读"：**同一事务内多次读同一条数据，结果一致**（哪怕别的事务改了并提交了）

打个比方：线程A在一张纸上写了字，但还没把纸交给老师（提交），线程B去看的时候拿到的还是那张没写字的旧纸。等A把纸交了（提交），B再去看才能看到。

>  `@Transactional` 机制和 `synchronized` 机制**作用域不匹配**导致：锁先释放，事务还没释放。导致别的线程查到了旧数据，依旧会出现超卖现状

解决了锁谁的问题后，依然要解决`@Transactional` 机制和 `synchronized` 机制**作用域不匹配**问题

~~~java
 public Result sekillVoucher(Long voucherId) {
        //注入ISeckillVoucherService,通过id获取秒杀卷信息
        SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
        // 1.判断秒杀是否开始或结束
        if (seckillVoucher.getBeginTime().isAfter(LocalDateTime.now())) {
            return Result.fail("秒杀尚未开始");
        }
        if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())) {
            return Result.fail("秒杀已经结束");
        }
        // 2.判断库存是否充足
        if (seckillVoucher.getStock()<1) {
            //库存不足
            return Result.fail("库存不足！");
        }
     
        return this.createVoucherOrder(voucherId);
    }
@Transactional
public  Result createVoucherOrder(Long voucherId) {

	Long userId = UserHolder.getUser().getId();
    synchronized (userid.toString().intern()){
        // 5.1.查询订单
        int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
        // 5.2.判断是否存在
        if (count > 0) {
            // 用户已经购买过了
            return Result.fail("用户已经购买过一次！");
        }

        // 6.扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock = stock - 1") // set stock = stock - 1
                .eq("voucher_id", voucherId).gt("stock", 0) // where id = ? and stock > 0
                .update();
        if (!success) {
            // 扣减失败
            return Result.fail("库存不足！");
        }

        // 7.创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        // 7.1.订单id
        long orderId = redisIdWorker.nextId("order");
        voucherOrder.setId(orderId);
        // 7.2.用户id
        voucherOrder.setUserId(userId);
        // 7.3.代金券id
        voucherOrder.setVoucherId(voucherId);
        save(voucherOrder);

        // 7.返回订单id
        return Result.ok(orderId);
    }     
}
~~~

![image-20260805220238365](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260805220238365.png)

<h4>回到当前代码</h4>

1. 线程A执行 `save()`，订单写进数据库了
2. 线程A还没提交事务，这时释放了锁
3. 线程B拿到锁，执行 `count` 查询 → **查到的是旧数据，看不到A刚写的那条** → count=0
4. 线程B认为没买过，又下了一单
5. 两个事务都提交，数据库里变成两条

**所以问题不是"没写进去"，而是"写了但别人暂时看不到"。锁释放得太早，B在A提交之前就去查了，查到的是旧数据，骗过了一人一单的检查。**

> 事务机制我理解了，但为啥这种情况，事务提交的要比锁释放的晚呢？这种情况是必然还是偶然发生的呢？

因为**锁加在方法内部，事务包在方法外面**。执行顺序是固定的：

```
代理开启事务        ← 事务开始
  进入方法
  获取锁
  执行业务（查、扣、save）
  释放锁             ← 锁在这里释放
  方法返回
代理提交事务          ← 事务在这里才提交
```

锁在方法内的 synchronized 块结束时释放，事务要等方法返回后代理才提交。中间隔着"方法返回"这一步，所以锁总是先释放、事务后提交。这是位置决定的，不是谁快谁慢的问题。

<h4>必然还是偶然</h4>

分两件事看：

**"锁释放早于事务提交"——必然。** 每次执行都是这个顺序，因为锁在方法内、事务在方法外，结构固定，改不了。

**"会不会真的导致重复下单"——概率性的。** 只有在"锁释放到事务提交"这个很短的窗口里，恰好有别的线程进来查订单，才会出问题。如果这个窗口里没人来查，就不会出问题。所以并发越高、请求越密集，越容易撞上这个窗口出 bug。

一句话：**顺序是必然的（结构决定），出bug是偶然的（看并发时机），但只要这个窗口存在，高并发下迟早会撞上。** 所以这段代码不能用，必须把锁提到事务外面去。

> 将锁提到外面去后，事务依然没有生效。需要获得原始的事务对象操作才能生效（SpringAOP机制导致）

将锁提到事务外面后，解决了作用域问题，但由于SpringAOP机制导致这个事务并没有生效

~~~java
        Long userid = UserHolder.getUser().getId();
        synchronized (userid.toString().intern()){
            return this.createVoucherOrder(voucherId);
        }
~~~

`@Transactional` 是基于 **Spring AOP 代理**实现的，而 `this.createVoucherOrder()` 是**类内部自调用**，直接走原始对象，==绕过了代理==，注解根本没机会生效。

==这张图揭示了本质==

![image-20260805222803357](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260805222803357.png)

<h4>具体解释：</h4>

Spring 的 `@Transactional` 是基于 **AOP 动态代理**实现的——Spring 给你的类包了一层 Proxy 外壳，只有调用**穿过这层 Proxy** 时，Proxy 才会拦截请求、开启事务、执行方法、提交/回滚。

但 `this.createVoucherOrder()` 是在原始对象**内部**直接调用的，`this` 指向的是原始对象本身，不经过 Proxy。所以 `createVoucherOrder` 上的 `@Transactional` 根本没机会被读取，事务就不会开启。

如果 `createVoucherOrder` 里扣减库存成功、但保存订单失败，**库存扣减不会回滚**——这就是事务失效的表现。

<h4>三种解法</h4>

<h5>方案一：注入自己（最简单，改动最小）</h5>

```java
@Service
public class VoucherOrderServiceImpl ... {
    // 注入自己的代理对象
    @Resource
    private IVoucherOrderService voucherOrderService;

    @Override
    public Result sekillVoucher(Long voucherId) {
        // ...
        synchronized (userid.toString().intern()){
            return voucherOrderService.createVoucherOrder(voucherId);  // ← 用代理调用，不用 this
        }
    }
}
```

注入的是 `IVoucherOrderService` 接口类型，Spring 给你的是**代理对象**，调用穿过 Proxy → 事务生效。

<h5>方案二：AopContext 获取当前代理</h5>

```java
synchronized (userid.toString().intern()){
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.createVoucherOrder(voucherId);
}
```

1.需要加入依赖`spring-boot-starter-aop`

~~~java
        <!-- 引入AOP能力 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
~~~

2.需要在启动类加 `@EnableAspectJAutoProxy(exposeProxy = true)`。//引入spring-boot-starter-aop会默认开启

~~~java
@EnableAspectJAutoProxy(exposeProxy = true)
@MapperScan("com.hmdp.mapper")
@SpringBootApplication
public class HmDianPingApplication {

    public static void main(String[] args) {
        SpringApplication.run(HmDianPingApplication.class, args);
    }

}
~~~

tip:教程单独引了 `aspectjweaver`，**`spring-boot-starter-aop` = `spring-aop` + `aspectjweaver` + 自动配置，是 Spring Boot 项目里用 AOP 功能的标准入口。**

<h5>方案三：拆到另一个 Service（最规范）</h5>

把 `createVoucherOrder` 移到一个新的 Service 类里，通过注入调用。职责分离，彻底避免自调用。

> 对SpringAOP机制的补充（上个问题看不懂可以看看这个补充）：什么是 Proxy 边界？

Spring 启动时，发现你的 `VoucherOrderServiceImpl` 上面有 `@Transactional` 注解的方法，就会给它**包一层外壳**——这个外壳就是 **Proxy（代理对象）**。

关键在于：**事务的拦截逻辑（开启事务、提交、回滚）写在 Proxy 这个外壳里，不在你的原始类里。**

![Spring_AOP代理边界原理](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/Spring_AOP%E4%BB%A3%E7%90%86%E8%BE%B9%E7%95%8C%E5%8E%9F%E7%90%86.png)

"穿过 Proxy 边界"就是——**调用方的引用指向的是 Proxy 对象，而不是原始对象**。外部注入的是 Proxy，所以外部调用会穿过；`this` 指向原始对象，所以 `this.xxx()` 不穿过。事务拦截器只住在 Proxy 里，没穿过它，它就不知道你在调什么方法。

> 用锁锁住了整个方法，那方法中乐观锁还有意义吗？

首先问的本身就有误区——==**这里的 `synchronized` 锁的不是整个方法，锁的是 `userid`**。==

先说结论：**有意义，因为这两个锁解决的根本不是同一个问题。**

![image-20260806093632939](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260806093632939.png)

关键在于——**你的 `synchronized` 锁的不是整个方法，锁的是 `userid`**。

看你第 58 行：

```java
synchronized (userid.toString().intern()) {
```

这意味着：

- **用户 A 的两个线程** → 同一个 userid → 串行，排队等锁
- **用户 A 和用户 B** → 不同 userid → **不互斥，并行执行** 

<h4>所以乐观锁防的是什么</h4>

假设库存只剩 1 件，用户 A、B、C、D 同时下单：

```
synchronized 的世界里：
  用户A 拿到锁 → 执行 createVoucherOrder → 释放锁
  用户B 拿到锁 → 执行 createVoucherOrder → 释放锁
  用户C 拿到锁 → 执行 createVoucherOrder → 释放锁
  用户D 拿到锁 → 执行 createVoucherOrder → 释放锁

  ↑ 四个用户各自持不同的锁，全部并行进入 createVoucherOrder
  ↑ 同时执行 stock=stock-1，没有 where stock>0 → 库存变 -3，超卖！
```

==`synchronized(userid)` 只保证"同一个用户不会重复下单"，但**不同用户的请求是并行的**==，它们会同时执行到第 78-81 行的扣库存 SQL。如果没有 `gt("stock", 0)` 这个条件，4 个人都能扣成功，库存变成 -3。

<h4>两个锁各管各的</h4>

|                    | synchronized(userid)         | where stock > 0            |
| :----------------- | :--------------------------- | :------------------------- |
| **防什么**         | 同一用户重复下单（一人多单） | 不同用户同时扣库存（超卖） |
| **锁粒度**         | 单个用户                     | 库存值                     |
| **不同用户互斥？** | 否，并行                     | 是，数据库行锁             |
| **集群有效？**     | 否，JVM 级别                 | 是，数据库级别             |

还有一层——即使单机环境下 synchronized 碰巧能串行化所有用户（如果锁的是全局对象），==**集群部署**时多台服务器各有各的 JVM，synchronized 完全失效==，数据库的乐观锁是最后的防线。

**一句话：synchronized 管"一人一单"，乐观锁管"不超卖"，两个问题，两把锁，缺一不可。**

### 整体学完后的回顾疑问

> 为什么数据库数据已近更新了，但由于事务提交，导致别的线程还是查到了旧值，此时别的线程为啥不能直接查到数据库的值？
> 这是由于事务的快照机制导致的？那假设我去掉事务，此时别的线程b是不是立马就能查到a线程改过的值了？

#### 一、先对齐场景

```java
@Transactional
public Result createVoucherOrder(Long voucherId) {
    // ① 查询订单：count = query().eq("user_id", userid).eq("voucher_id", voucherId).count();
    if (count > 0) return Result.fail("已经买过");
    
    // ② 扣库存
    boolean success = seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId).gt("stock", 0)
        .update();
    
    // ③ 创建订单
    save(voucherOrder);
}
```

两个线程 A、B 同时抢同一张券（同一用户）：

- A 先开始事务，查 count=0，没下过单，准备往下走
- B 也开始事务，**查 count 也=0**（A 还没提交）
- A、B 各自扣库存、各自保存订单 → **重复下单**

用户问的"数据库数据已经更新了但别的线程还是查到了旧值"——指的就是 B 看到的那个 count=0。

#### 二、为什么 B 看到的是旧值——**MVCC + REPEATABLE READ 快照机制**

MySQL InnoDB 默认隔离级别是 **REPEATABLE READ（可重复读）**，它的核心机制叫 **MVCC（多版本并发控制）**，每个事务都会有一份"快照"：

plaintext

```
时间线 ─────────────────────────────────────────────────────►

       事务B开始           A提交事务              B查count
         │                  │                    │
         ▼                  ▼                    ▼
   ┌────────────────┐    ┌────────┐    ┌────────────────┐
   │ B 拍快照       │    │ 数据库 │    │ B 查 count     │
   │ (看到旧版本)   │    │ 已更新 │    │ 走快照，仍是0  │
   └────────────────┘    └────────┘    └────────────────┘
```

**关键规则**：

1. **快照在事务第一次 SELECT 时建立**（更精确：第一次"快照读"时），之后这个事务内所有的**普通 SELECT** 都读这个快照版本
2. **A 的提交对 B 的快照不可见**——A 改了数据，但 B 的事务从一开始就读旧版本，A 提交后 B 的快照不会被刷新
3. 这就是 REPEATABLE READ 的"可重复读"：**同一事务内多次读同一条数据，结果一致**（哪怕别的事务改了并提交了）

> 类比：你在图书馆复印了一份某本书的副本（事务开始拍快照）。后来这本书被原作者修订了第 50 页（其他事务提交）。但**你的副本不变**，你看到的内容永远是复印那一刻的。

#### 三、那么去掉事务会怎么样？——**能立刻看到**，但带来新问题

##### 不加 `@Transactional`（自动提交模式）

```java
public Result createVoucherOrder(Long voucherId) {  // 无 @Transactional
    int count = query()....count();  // 自动提交，每次 SELECT 都是独立事务，读最新已提交
    if (count > 0) return ...;
    // ...
    save(voucherOrder);  // 自动提交
}
```

每条 SQL 各自独立提交，**每次 SELECT 都是新事务**——直接读数据库最新已提交版本，没有"快照隔离"。

**那 B 是不是立刻能看到 A 的修改？**

- **如果 B 的 SELECT 在 A 的 INSERT 提交之后执行** → ✅ 能看到，B 不会再插 → 不重复
- **如果 B 的 SELECT 在 A 的 INSERT 提交之前执行** → ❌ 仍然看到 count=0，B 还是会往下走 → 仍然重复

**所以去掉事务只是解决了"看到旧值"这个问题，并没有解决"同时抢"的并发问题**。这俩是两件事：

| 维度           | 事务机制（MVCC + RR）         | 并发控制                       |
| :------------- | :---------------------------- | :----------------------------- |
| 解决什么       | "看到的是哪个版本的数据"      | "两个线程同时改会不会冲突"     |
| 不加事务会怎样 | 每条 SQL 都是新事务，看到最新 | 两个线程还是能同时进入 if 判断 |

#### 四、一句话总结

**B 看到旧值是 MySQL InnoDB 的 MVCC 快照机制造成的**：B 的事务从一开始就有自己的快照，A 即使提交了也对 B 不可见，所以 B 看到的还是 A 提交前的"无人下单"状态。**去掉事务确实能让 B 看到 A 提交后的新值（每次 SELECT 都是新事务，没有快照）**，但这只是解决了"可见性"，**没有解决"两个线程同时进入 if 分支"的并发竞争**——这需要锁（synchronized/分布式锁）或原子操作（Redis Lua）来保证 `check + write` 的不可分割性。黑马点评从版本 1 演进到版本 3 的整个过程，就是在分别解决这两个层面的问题。

#### 五、实战补充：**双保险才是生产做法**

==真实项目里，`(user_id, voucher_id)` 上通常会建**唯一索引**：==

sql

```sql
UNIQUE KEY uk_user_voucher (user_id, voucher_id)
```

这样即使上面的所有并发控制都失效（锁丢了、Lua 没拦住），最后 `INSERT` 时数据库会直接报 `DuplicateKeyException`——这是**最后一道防线**，成本极低但收益极大。

## 分布式锁

### 实战篇11（分布式锁解决集群部署，synchronized失效问题）

> 为什么集群部署情景下，这里的synchronized开始失效了呢？

底层原因在于==不同的服务器有自己独立的jvm环境，不同的jvm中有各自的锁监视器对象来监视线程==。他们的代码写的虽然和服务器A一样，但是==锁监视器对象却不是同一个==，所以线程3和线程4可以实现互斥，但是却无法和线程1和线程2实现互斥，这就是 集群环境下，syn锁失效的原因，在这种情况下，我们就需要使用分布式锁来解决这个问题。

![1653374044740](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/1653374044740.png)

### 实战篇13（分布式锁误删）

>`ID_PREDIX`不是static类型的吗？属于类级别类对象只有独一份，所有对象共享这一份啊？为啥ID_PREDIX+Thread.currentThread().getId()可以用来标识不同的线程呢？

```java
...
private static final String ID_PREDIX= UUID.randomUUID().toString();

@Override
public boolean tryLock(long timeoutSec) {
    //获取线程标识
    String threadId = ID_PREDIX+Thread.currentThread().getId();
...
```

==理解的概念没错，但那都针对的是单个JVM来说，但这里的场景是跨JVM==

你的疑问涉及几个容易混淆的概念，逐个理清。

<h4>1. static 和非 static 成员变量的区别</h4>

```java
public class SimpleRedisLock {
    private String name;                              // 实例变量（非static）
    private static final String KEY_PREFIX = "lock:";  // 类变量（static）
    private static final String ID_PREDIX = UUID.randomUUID().toString();  // 类变量（static）
}
```

| 类型     | 关键字    | 存储位置           | 数量             | 谁能访问           |
| :------- | :-------- | :----------------- | :--------------- | :----------------- |
| 实例变量 | 无 static | 每个对象内部       | 每个对象各有一份 | 通过对象访问       |
| 类变量   | static    | 方法区（类信息里） | **全类只有一份** | 通过类名或对象访问 |

`name` 是实例变量 — 你 `new` 3 个 SimpleRedisLock 对象，就有 3 个 `name`，互不影响。

==对于同一个JVM来说：==`ID_PREDIX` 是 `static` 类变量 — 不管你 `new` 多少个对象，**全类只有一份**，所有对象共享。

<h4>2. "类对象只有独一份"是什么意思？</h4>

你之前了解的没错，但说的是另一回事：

```
JVM 加载 SimpleRedisLock.class → 在方法区生成一个 Class 对象
                                  ↑ 这就是"类对象只有独一份"
                                  ↑ static 字段就存在这里
```

**类对象（Class 对象）**是 JVM 用来描述一个类的元数据结构，每个类只有一个。`static` 字段就挂在这个类对象上。

**实例对象（你 new 出来的）**可以有无数个，每个实例对象内部各有一份非 static 字段。

用图来表示：

```
方法区（只有一份）
┌─────────────────────────────────────┐
│  SimpleRedisLock 的 Class 对象       │
│  ┌─────────────────────────────┐    │
│  │ KEY_PREFIX = "lock:"         │    │  ← static，全类共享
│  │ ID_PREDIX  = "a3b2c1d4-..."  │    │  ← static，全类共享
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘

堆（每个 new 一份）
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 实例对象 1    │  │ 实例对象 2    │  │ 实例对象 3    │
│ name="order:1"│  │ name="order:2"│  │ name="order:3"│  ← 非static，各管各的
│ stringRedis..│  │ stringRedis..│  │ stringRedis..│
└──────────────┘  └──────────────┘  └──────────────┘
```

<h4>3. ID_PREDIX 是每个线程都有自己的吗？</h4>

**不是。** `ID_PREDIX` 是 `static` 的，整个 JVM 里就一份，所有线程读到的值都一样。

真正区分线程的是后面拼的 `Thread.currentThread().getId()`：

```java
String threadId = ID_PREDIX + Thread.currentThread().getId();
//                 ↑ JVM级别唯一（所有线程相同）   ↑ 线程级别唯一（每个线程不同）
```

假设 `ID_PREDIX = "a3b2c1d4-..."`：

| 线程   | ID_PREDIX    | Thread.getId() | 拼接结果      |
| :----- | :----------- | :------------- | :------------ |
| 线程-1 | a3b2c1d4-... | 1              | a3b2c1d4-...1 |
| 线程-2 | a3b2c1d4-... | 2              | a3b2c1d4-...2 |
| 线程-3 | a3b2c1d4-... | 3              | a3b2c1d4-...3 |

<h4>4. 为什么需要 ID_PREDIX？</h4>

`Thread.currentThread().getId()` 只在**单个 JVM 内**保证唯一。但分布式场景下有多个 JVM 进程：

```
JVM-A (服务器1)          JVM-B (服务器2)
  线程-1 → getId()=1       线程-1 → getId()=1
  线程-2 → getId()=2       线程-2 → getId()=2
```

如果只用 `getId()`，JVM-A 的线程 1 和 JVM-B 的线程 1 标识都是 `"1"`，JVM-A 的线程 1 就可能删掉 JVM-B 的线程 1 的锁。

加上 `ID_PREDIX`（每个 JVM 启动时 `UUID.randomUUID()` 生成一个唯一值）：

```
JVM-A: ID_PREDIX = "aaa-111..."  → 线程1标识 = "aaa-111...1"
JVM-B: ID_PREDIX = "bbb-222..."  → 线程1标识 = "bbb-222...1"
```

这样即使两个 JVM 的线程 ID 相同，UUID 前缀不同，整体标识也不同，**跨 JVM 不会误删**。

<h4>总结


- **`static`** → 类级别，全类一份，所有对象共享，所有线程共享
- **非 static** → 实例级别，每个 `new` 出来的对象各有一份
- **`ID_PREDIX`** → static，整个 JVM 一份，不是每个线程一个，也不是每个对象一个
- **==同一JVM==线程唯一性** → 靠 `Thread.currentThread().getId()` 保证
- **==跨 JVM 唯一性==** → 靠 `ID_PREDIX`（UUID）保证
- 两者拼接 → 全球唯一线程标识

### 实战篇16（java调用lua脚本）

> unlock.lua讲解

```lua
-- KEYS[1] = 锁的key，ARGV[1] = 当前线程标识

if(redis.call('GET',KEYS[1])==ARGV[1])then    -- 从Redis取出锁的value，和当前线程标识比较
    return redis.call('DEL',KEYS[1])           -- 一致 → 删除锁，返回删除结果
end
return 0;                                       -- 不一致 → 什么都不做，返回0
```

逐行标注：

| 代码                        | 作用                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| `redis.call('GET',KEYS[1])` | 调用 Redis 的 GET 命令，取出锁 key 当前存的 value（即加锁时写入的线程标识） |
| `== ARVG[1]`                | 和传入的当前线程标识做比较，判断"这把锁是不是我的"           |
| `redis.call('DEL',KEYS[1])` | 如果是我的锁，调用 DEL 删除这个 key，返回值为删除的数量（1=成功） |
| `return 0`                  | 如果不是我的锁，直接返回 0，什么都不删                       |

> SimpleRedisLock中lua相关代码

<h4>1.静态初始化 — 类加载时加载脚本（第 24-29 行）</h4>

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
static{
    UNLOCK_SCRIPT=new DefaultRedisScript<>();              // 创建脚本对象
    UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua")); // 从classpath读取unlock.lua
    UNLOCK_SCRIPT.setResultType(Long.class);               // 脚本返回值类型是Long
}
```

| 代码                              | 作用                                                         |
| :-------------------------------- | :----------------------------------------------------------- |
| `DefaultRedisScript<Long>`        | Spring 提供的 Lua 脚本封装，`Long` 是返回值类型（对应 Lua 的 `return redis.call('DEL',...)` 返回的数字） |
| `static { }`                      | 静态代码块，**类加载时执行一次**，脚本只读取一次             |
| `ClassPathResource("unlock.lua")` | 从 classpath（`src/main/resources/`）下找 `unlock.lua` 文件  |
| `setLocation(...)`                | 告诉 Spring 脚本文件在哪                                     |
| `setResultType(Long.class)`       | 告诉 Spring 脚本返回值是 Long 类型，Spring 会据此做类型转换  |

==**为什么用 static 加载而不是每次 unLock 时读文件？** 脚本内容是固定的，类加载时读一次就够了。每次释放锁都读文件是浪费 IO。==

<h4>2. 执行脚本 — unLock 方法（第 40-45 行）</h4>

```java
@Override
public void unLock() {
    stringRedisTemplate.execute(UNLOCK_SCRIPT,                        // 执行预加载的Lua脚本
            Collections.singletonList(KEY_PREFIX + name),              // KEYS 列表：只有一个元素，锁的key
            ID_PREFIX + Thread.currentThread().getId());              // ARGV 列表：只有一个元素，当前线程标识
}
```

| 参数                                           | 对应 Lua 中的 | 值                  | 作用               |
| :--------------------------------------------- | :------------ | :------------------ | :----------------- |
| `UNLOCK_SCRIPT`                                | 整个脚本      | 预加载的 unlock.lua | 要执行的脚本       |
| `Collections.singletonList(KEY_PREFIX + name)` | `KEYS[1]`     | `"lock:order:1001"` | 锁的 Redis key     |
| `ID_PREFIX + Thread.currentThread().getId()`   | `ARGV[1]`     | `"a3b2c1d4-...42"`  | 当前线程的唯一标识 |

==`Collections.singletonList` 生成一个不可变的单元素 List，因为这里只有一个 key。如果有多个 key，就用 `Arrays.asList(k1, k2, ...)`。==

<h4>3. 对比旧版被注释掉的代码（第 47-59 行）</h4>

```java
// 旧版：两条独立的 Redis 命令
String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);  // 第1步：GET
if (threadId.equals(id)) {
    stringRedisTemplate.delete(KEY_PREFIX+name);                        // 第2步：DEL
}
```

旧版的问题就在这两步之间 — **不是原子操作**：

```
线程A                          线程B
GET lock → "A的标识" ✓
                              (锁刚好过期自动删除)
                              SET lock "B的标识" (B抢到锁)
DEL lock (删了B的锁！)
```

Lua 脚本在 Redis 中是**原子执行**的 — Redis 执行脚本期间不会执行其他命令，GET 判断和 DEL 之间不会被任何其他操作插入，彻底解决了误删问题。

## 分布式锁-Redisson

### 实战篇17（Redisson功能介绍）（解决可重入、可重试、超时释放、主从一致性）

==分布式锁章节自己实现了一个简单的分布式锁，但有四个问题需要解决：==

- 可重入
- 可重试
- 超时释放
- 主从一致性

#### 问题一：可重入

```java
// 线程A 调用 methodA()
public void methodA() {
    lock.tryLock("order:123");  // 第1次加锁，成功
    methodB();              // 调用内部方法
    lock.unlock("order:123");
}

public void methodB {
    lock.tryLock("order:123");  // 第2次加锁，同一个线程，同一个key
    // ↑ SETNX 发现 key 已存在，返回 false，加锁失败
    // 但这把锁明明是线程1自己加的
    ...
}
```

**问题**：同一个线程，第二次拿同一把锁，失败了。`SETNX` 只看 key 在不在，不看是谁的。

![image-20260810174638422](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810174638422.png)

#### 问题二：可重试

==之前提到的自旋操作其实也是解决可重试的方案（主动轮询）。redison提供的pub/sub方案（被动通知）同样解决可重试。两者思想不同但解决的是同样的问题==

```java
// 100个线程同时抢锁
// T1 抢到了，执行业务需要 5ms
// T2 在第 2ms 时尝试加锁：

boolean ok = lock.tryLock("order:123");  // SETNX 失败，返回 false
// 直接 return false，不等待
// T3、T4...T100 同理，全部失败

// 实际上 T1 在第 5ms 就释放了锁，T2~T100 只要等 3ms 就能拿到
// 但 tryLock 不等，立刻返回 false
```

**问题**：抢锁失败就立刻返回 false，不等。哪怕锁 3ms 后就释放了，也不试第二次。

![image-20260810174659439](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810174659439.png)

#### 问题三：超时释放

```java
// 场景A：设 10 秒过期
lock.tryLock("order:123", 10);  // TTL = 10s
// 业务执行了 15 秒
// 第 10 秒：锁过期自动删除
// 第 10 秒：线程B 加锁成功
// 第 10~15 秒：线程A 和线程B 同时执行业务 → 超卖

// 场景B：设 60 秒过期
lock.tryLock("order:123", 60);  // TTL = 60s
// 第 5 秒：进程宕机
// 锁要等到第 60 秒才过期
// 其他线程白等 55 秒
```

**问题**：过期时间是写死的。设短了业务没跑完锁就没了，设长了进程挂了别人干等。

![image-20260810174743500](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810174743500.png)

#### 问题四：主从一致性

```java
时间线：
t=0ms    线程A 在 master 加锁成功（master 内存里有锁数据）
         master 准备异步同步给 slave（还没发）
t=50ms   master 宕机，锁数据跟着 master 一起没了
         slave 上没有这把锁
t=200ms  slave 被提升为新 master
t=210ms  线程B 在新 master 上加锁
         新 master 上没有锁记录 → 加锁成功
         线程A 和线程B 同时持锁
```

**问题**：主从同步是异步的。master 加锁成功后还没同步给 slave 就宕机了，锁数据丢失。新 master 上查不到这把锁，别的线程能再加一把。

![image-20260810174800351](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810174800351.png)

### 实战篇19（Redisson解决可重入）

<h4>Java 的 ReentrantLock 是怎么解决的可重入</h4>

`ReentrantLock` 内部维护了一个**计数器**：

```
线程A 第一次获取锁 → 计数器 = 1，记录持有者是线程A
线程A 第二次获取锁 → 发现持有者就是自己 → 计数器 = 2，直接放行
methodB 执行完，释放锁 → 计数器 = 1（还没到0，不真正释放）
methodA 执行完，释放锁 → 计数器 = 0（真正释放锁）
```

<h4>你的 Redis 锁要实现可重入需要什么</h4>

Redis 里没法直接用计数器，需要用 **Hash 结构** 代替：

```
之前（不可重入）：
  key: lock:order
  value: "uuid-1"              ← String类型，只能存一个值

改成（可重入）：
  key: lock:order
  value: Hash {
    field: "uuid-1"            ← 线程标识
    value: 2                   ← 重入次数
  }
```

加锁逻辑（Lua 脚本）：

1. key 不存在 → 创建 Hash，field=线程标识，value=1，设过期时间
2. key 存在且 field=当前线程标识 → value + 1（重入）
3. key 存在但 field≠当前线程标识 → 加锁失败

释放锁逻辑（Lua 脚本）：

1. key 不存在 或 field≠当前线程标识 → 直接返回
2. field=当前线程标识 → value - 1
3. value 减到 0 → DEL key（真正释放）
4. value > 0 → 不删，还有重入没退完

==这个可以用lua试一下能不能实现可重入，教程建议直接用现成的Redission框架直接提供这些解决方案==

### 实战篇20（Redisson解决可重试和超时释放）

#### 可重试-解决思路

失败后不立刻返回，而是**订阅锁的释放消息 + 信号量阻塞等待**：

```
旧方案：tryLock 失败 → return false（一锤子买卖）

新方案：
  1. 加锁失败 → 订阅 Redis 频道 lock:order:123
  2. 信号量阻塞，挂起线程（不占 CPU）
  3. 持锁线程 unlock → publish 释放消息
  4. 订阅回调收到消息 → 释放信号量 → 唤醒等待线程
  5. 等待线程被唤醒 → 重新执行 Lua 加锁
```

**一句话**：不轮询，等通知。锁一释放，Redis 主动推消息把你叫醒。

<h4>源码实现（结合课程讲解看源码）</h4>

```java
// RedissonLock.java — tryLock(waitTime, leaseTime, unit)
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    return tryLock(waitTime, -1, unit);
}

private boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();

    // 第1步：尝试加锁（执行上面那段 Lua 脚本）
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    if (ttl == null) {
        return true;  // 加锁成功，直接返回
    }

    // 加锁失败，计算剩余等待时间
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;  // 等待时间已用完
    }

    // 第2步：订阅锁的释放消息频道
    current = System.currentTimeMillis();
    RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    // ... 等待订阅成功 ...

    // 第3步：订阅成功后，再试一次加锁
    ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    if (ttl == null) {
        return true;  // 第二次尝试成功
    }

    // 第4步：仍然失败，用信号量阻塞等待，直到收到释放消息或超时
    if (ttl >= 0) {
        // entry.getLatch() 返回一个 Java Semaphore
        // tryAcquire(ttl) 阻塞等待，最多等 ttl 毫秒
        entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
    }

    // 第5步：被唤醒后，再次尝试加锁（循环回到第3步）
    // 如果超过总等待时间 waitTime，返回 false
    ...
}
```

订阅回调收到释放消息时的处理（`RedissonLockEntry`）：

```java
// 当收到 publish 消息时，执行：
@Override
public void onMessage(String channel, Object message) {
    if (message.equals(unlockMessage)) {
        // 释放信号量，唤醒等待的线程
        semaphore.release();
    }
}
```

核心链路：`tryLock 失败 → subscribe 频道 → semaphore.tryAcquire 阻塞 → unlock publish → onMessage → semaphore.release 唤醒 → 重新 tryAcquire`。

![image-20260810204552500](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810204552500.png)

#### 超时释放-解决思路（看门狗）

==默认不传入过期时间走看门狗机制，手动传入过期时间依然是原始的到期就死==

不设死过期时间，改成**定期续命**：

```
旧方案：SET key value EX 10  → 10 秒后必死，不管业务跑没跑完

新方案（看门狗）：
  1. 加锁时设 TTL = 30s（默认 lockWatchdogTimeout）
  2. 同时启动一个定时任务，每 10s（= 30s / 3）跑一次续期
  3. 续期：Lua 脚本判断 hget == 自己的标识，是就 pexpire 重置为 30s
  4. 线程活着 + 没 unlock → 永远续命
  5. 线程宕机 → 定时任务随之消亡 → 30s 后锁自动释放
```

**一句话**：业务没跑完就续命，进程挂了就放手。用"续"代替"猜"。

<h4>源码实现(结合课程看源码)</h4>

```java
// RedissonLock.java — 加锁成功后启动看门狗
private void scheduleExpirationRenewal(long threadId) {
    // 创建一个 RenewalTask，存入 ConcurrentHashMap 防止重复启动
    ExpirationEntry entry = new ExpirationEntry();
    ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
    if (oldEntry != null) {
        oldEntry.addThreadId(threadId);
    } else {
        entry.addThreadId(threadId);
        // 真正启动定时续期任务
        renewExpiration();
    }
}

private void renewExpiration() {
    ExpirationEntry entry = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (entry == null) {
        return;
    }

    // 创建 Netty 时间轮定时任务
    Timeout task = commandExecutor.getConnectionManager().getGroup().schedule(() -> {
        ExpirationEntry e = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (e == null) {
            return;
        }
        Long threadId = e.getFirstThreadId();
        if (threadId == null) {
            return;
        }

        // 执行续期 Lua 脚本
        RFuture<Boolean> future = renewExpirationAsync(threadId);
        future.onComplete((res, ex) -> {
            if (ex != null) {
                log.error("续期失败", ex);
                return;
            }
            if (res) {
                // 续期成功，递归调用自己，安排下一次续期
                renewExpiration();
            }
            // res == false 说明锁已经不是自己的了（已释放或被别人抢），停止续期
        });
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
    // ↑ internalLockLeaseTime = 30000ms，除以 3 = 每 10000ms（10秒）续期一次

    entry.setTimeout(task);
}
```

续期的 Lua 脚本：

```lua
-- KEYS[1] = 锁 key
-- ARGV[1] = 过期时间（30000ms）
-- ARGV[2] = 线程标识

if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    -- 锁还在，且是当前线程的，重置过期时间
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return 1;
else
    -- 锁不是当前线程的了（已释放），停止续期
    return 0;
end;
```

解锁时取消看门狗：

```java
// RedissonLock.java — unlock 时取消续期任务
public RFuture<Void> unlockAsync(long threadId) {
    return unlockInnerAsync(threadId).onComplete((res, e) -> {
        // 取消定时任务，从 map 中移除
        cancelExpirationRenewal(threadId);
    });
}
```

三个关键点：定时周期 = TTL / 3；续期前先 `hexists` 校验身份；unlock 时取消定时任务。

![image-20260810211525053](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810211525053.png)

### 实战篇21（Redisson解决主从不一致）

#### 主从一致性（MultiLock / RedLock）-解决思路

> 课程讲的是最开始的MultiLock，要求所有锁成功才能加锁成功，RedLock要求过半成功即可显然更优秀。

==1、RedissonRedLock 继承自 RedissonMultiLock，这里给的例子是RedLock，但两者只是判断阈值不同，底层原理是相同的。==

==2、联锁\红锁用于加锁的redis节点都是master节点，不存在主从之分。但这些mater节点自己可以有各自的从节点==

<h4>具体原理</h4>

不依赖单个节点，改成**多节点过半表决**==RedLock==：

```
旧方案：在 1 个 master 上加锁 → master 宕机 → 锁丢失

新方案（RedLock）：
  1. 准备 N 个独立的 Redis 实例（不是主从，是独立部署）
  2. 依次向每个实例发 SETNX 加锁
  3. 成功数 >= N/2 + 1（过半）且总耗时 < TTL → 加锁成功
  4. 不满足 → 加锁失败，回滚所有已加锁的实例
```

**一句话**：不把鸡蛋放一个篮子里，过半数同意才算数，少数节点挂了不影响。

<h4>源码实现</h4>

```java
// RedissonMultiLock.java — 多节点加锁入口
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    // tryLockCount = 所有参与的锁实例数量（如 5 个 RedissonLock）
    // locks = 多个 RedissonLock 实例数组

    List<RFuture<Boolean>> futures = new ArrayList<>();
    for (RLock lock : locks) {
        // 逐个向每个 Redis 实例发起异步加锁
        futures.add(lock.tryLockAsync(waitTime, leaseTime, unit));
    }

    // 等待所有加锁结果返回
    int successCount = 0;
    for (RFuture<Boolean> future : futures) {
        if (future.getNow()) {
            successCount++;  // 统计成功数
        }
    }

    // 判定：成功数 >= N/2 + 1（过半）
    int majority = locks.size() / 2 + 1;
    if (successCount >= majority) {
        return true;   // 过半成功，加锁成功
    }

    // 未过半，回滚已加锁的实例
    for (int i = 0; i < futures.size(); i++) {
        if (futures.get(i).getNow()) {
            locks.get(i).unlockAsync();  // 逐个解锁
        }
    }
    return false;
}
```

==RedissonRedLock 继承自 RedissonMultiLock，区别只是判定阈值：==

```java
// RedissonRedLock.java — RedLock 就是 MultiLock 的过半版本
public class RedissonRedLock extends RedissonMultiLock {
    public RedissonRedLock(RLock... locks) {
        super(locks);
    }

    // RedLock 的核心：过半成功才算加锁成功
    @Override
    protected int failedLocksLimit() {
        return locks.size() - (locks.size() / 2 + 1);  // 允许失败的数量
        // 5 个节点 → 允许失败 2 个 → 至少 3 个成功
    }
}
```

实际使用：

```java
// 创建 3 个独立的 Redisson 客户端（连 3 个独立 Redis 实例）
Config config1 = new Config(); config1.useSingleServer().setAddress("redis://192.168.1.1:6379");
Config config2 = new Config(); config2.useSingleServer().setAddress("redis://192.168.1.2:6379");
Config config3 = new Config(); config3.useSingleServer().setAddress("redis://192.168.1.3:6379");

RedissonClient client1 = Redisson.create(config1);
RedissonClient client2 = Redisson.create(config2);
RedissonClient client3 = Redisson.create(config3);

// 各自获取锁对象
RLock lock1 = client1.getLock("lock:order:123");
RLock lock2 = client2.getLock("lock:order:123");
RLock lock3 = client3.getLock("lock:order:123");

// 用 RedLock 包装
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);

// 加锁：3 个中至少 2 个成功才算成功
boolean ok = redLock.tryLock(10, TimeUnit.SECONDS);
```

![image-20260810210647609](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260810210647609.png)

#### MultiLock和**RedLock**

> MultiLock和**RedLock**区别？

<h4>一句话总结</h4>

| 方案        | 成功条件     | 容错   | 解决主从一致性？ |
| :---------- | :----------- | :----- | :--------------- |
| MultiLock   | 全部成功     | 无     | 不解决，更脆弱   |
| **RedLock** | **过半成功** | **有** | **解决**         |

解决主从一致性的核心就是**过半表决**——不依赖任何单个节点，也不要求所有节点，少数挂了照样能加锁，且数学上保证互斥。

> 那更脆弱了MultiLock的意义何在呢？

MultiLock 不是为了解决主从一致性，它和 RedLock 解决的是**完全不同的问题**。

<h4>MultiLock 的真正用途：同时锁多个资源 </h4>

MultiLock 的场景是：一个操作需要**同时持有多个不同的锁**，要么全拿到，要么一个都不要。

```java
场景：转账，从账户 A 转到账户 B

需要同时锁住 A 和 B 两个账户，防止转账过程中有人修改余额。

RLock lockA = client.getLock("account:A");
RLock lockB = client.getLock("account:B");
RedissonMultiLock multiLock = new RedissonMultiLock(lockA, lockB);

multiLock.lock();  // A 和 B 都加锁成功才返回
// 执行转账：A 扣钱，B 加钱
multiLock.unlock(); // 同时释放
```

**为什么必须全部成功？** 如果只锁了 A 没锁到 B，你扣了 A 的钱，B 还没加钱，这时候有人改了 B 的余额，数据就错了。所以"全成功才算成功"是**业务要求**，不是缺陷。



<h4>RedLock 的用途：一个锁存多份防丢</h4>

RedLock 的场景是：锁**同一个资源**，但存在多个独立节点上防丢失。

```java
场景：秒杀扣库存，只需要锁 lock:stock:1001

5 个独立 Redis 实例，每个都存同一把锁。
过半成功 = 锁拿到了。
某个节点挂了不影响。
```

<h4>核心区别</h4>

|                | MultiLock                | RedLock                    |
| :------------- | :----------------------- | :------------------------- |
| **锁的是什么** | 多个**不同**资源         | 一个资源存多份             |
| **成功条件**   | 全部成功                 | 过半成功                   |
| **目的**       | 原子性：要么全锁要么不锁 | 容错性：少数节点挂了也能用 |
| **典型场景**   | 转账（锁 A 和 B）        | 防主从丢锁（锁存 5 份）    |

<h4>一句话</h4>

- **MultiLock**：我需要**同时拿好几把不同的锁**，少一把都不行（业务需要）
- **RedLock**：我只需要**一把锁**，但怕丢了所以存好几份，过半在就行（容灾需要）

所以 MultiLock "更脆弱"不是它的缺点，而是它的设计目的——它本来就不是用来容灾的，是用来保证多资源操作原子性的。

> MultiLock不就是像事务一样吗？那我为啥不用事务呢？

#### MultiLock 和事务：看着像，本质完全不同

你的直觉有道理——两者都追求"要么都成功，要么都不算数"。但 **MultiLock 解决的是"分布式互斥"，事务解决的是"数据一致性"**，目标不同，不能互相替代。

##### 一、先说结论

|                    | MultiLock（Redisson）                        | 数据库事务                            |
| :----------------- | :------------------------------------------- | :------------------------------------ |
| **本质**           | **分布式锁**：多把锁一起加，全部成功才算成功 | **ACID**：一组数据操作原子提交/回滚   |
| **保证什么**       | **互斥**：同一资源同一时刻只一个持有者       | **原子性 + 一致性 + 隔离性 + 持久性** |
| **操作对象**       | Redis 里的锁 key                             | 数据库表里的行/数据                   |
| **失败时**         | 释放已加的锁                                 | `ROLLBACK` 撤销已执行的 SQL           |
| **解决的核心问题** | 跨 JVM/跨服务**谁先动**                      | 数据**改没改对、改没改全**            |

**MultiLock 不是"事务"，它只是"把多个互斥点绑成一个整体"；事务不保证互斥，MultiLock 不保证数据回滚。** 两者是**互补**，不是**替代**。

##### 二、MultiLock 到底做了什么

```java
RLock lock1 = client.getLock("lock:order:100");
RLock lock2 = client.getLock("lock:inventory:5");
RLock lock3 = client.getLock("lock:user:42");
RedissonMultiLock multiLock = new RedissonMultiLock(lock1, lock2, lock3);
multiLock.lock();
try {
    // 同时持有三把锁
} finally {
    multiLock.unlock();
}
```



- **加锁**：依次（或并行）对 3 把锁加锁，**全部成功才算成功**；任一失败，**回滚已加的**（释放）。
- **解锁**：全部释放。
- **保证**：这期间**没有别的 MultiLock 能同时持有这三把锁**（互斥）。
- **不保证**：你的业务数据改对了、改全了、没被别的事务污染。

**它像"事务"的只是"全有或全无"这个提交语义**，但那是**锁的获取**层面的全有全无，不是**数据修改**层面的原子提交。

##### 三、为什么不能"用事务代替 MultiLock"

##### 场景：秒杀扣库存 + 建订单 + 扣用户额度

java

```java
// 用事务（单库）
@Transactional
public void seckill() {
    updateStock();      // 扣库存
    createOrder();      // 建订单
    updateUserQuota();  // 扣额度
}
```



事务能保证**这三步在同一个数据库里原子提交/回滚**。但它**保证不了互斥**：

- 100 个线程同时进来，事务**并发执行**，各自读到 stock=100，**照样超卖**（除非你加 `SELECT ... FOR UPDATE` 或 `stock>0` 条件更新，那又是行锁/乐观锁的范畴）。
- 事务解决的是"**改到一半失败要回滚**"，不是"**同一时刻只能一个人改**"。

**要互斥，必须加锁**（行锁、分布式锁、乐观锁）；**要数据原子性，必须用事务**。两者**各管一摊**。

##### 场景：跨服务/跨库

java

```java
// 订单服务扣库存（库存服务是另一个库/另一个服务）
orderService.createOrder();       // 库 A
inventoryService.deductStock();   // 库 B（远程调用）
```



- **事务管不了跨库/跨服务**（本地事务只在一个数据源内，分布式事务成本高、复杂）。
- **MultiLock 可以**：在订单服务和库存服务各自用同一组 Redis 锁做互斥，保证"同一订单/同一库存不会被并发改"。
- 但 MultiLock **也不保证两个库的数据原子回滚**——扣库存成功了、建订单失败了，MultiLock 只保证锁互斥，**不帮你回滚库存**。这仍需**分布式事务（Seata/TCC/消息补偿）**或**业务补偿**。

##### 四、为什么不能"用 MultiLock 代替事务"

MultiLock 能保证"同时只有一个线程持有这组锁"，但：

- **不保证数据原子性**：你持锁后改了 A 表成功、改 B 表失败，MultiLock 不会帮你回滚 A 表。要回滚，还得靠**数据库事务**（`@Transactional` 或手动 `rollback`）。
- **不保证隔离性**：持锁期间别的**不走这把锁**的代码照样能改数据，MultiLock 管不了"所有访问路径都加锁"。
- **不保证持久性**：Redis 锁本身会过期，和数据库持久化无关。

##### 五、两者的正确组合方式

生产里常见的是**"锁 + 事务"配合**，而不是二选一：

java

```java
multiLock.lock();                    // ① 分布式锁：保证互斥（谁先动）
try {
    transactionTemplate.execute(status -> {   // ② 本地事务：保证数据原子性（改对改全）
        updateStock();
        createOrder();
        return null;
    });
} finally {
    multiLock.unlock();              // ③ 释放锁
}
```



- **锁**：防止并发冲突（超卖、重复下单）。
- **事务**：保证持锁期间的多步数据修改要么全成、要么全回滚。
- **锁的粒度**：跨服务互斥用 MultiLock，单库原子性用事务。

##### 六、什么时候用 MultiLock，什么时候用事务

| 需求                                       | 用什么                               |
| :----------------------------------------- | :----------------------------------- |
| 多把锁"全有或全无"地互斥（跨 key、跨服务） | **MultiLock**                        |
| 同一数据库多表操作原子提交/回滚            | **事务**（`@Transactional`）         |
| 并发下防超卖/防重复                        | **锁**（分布式锁 / 行锁 / 乐观锁）   |
| 跨库跨服务数据一致                         | **分布式事务**（Seata/TCC/消息补偿） |
| 既要互斥又要数据原子                       | **锁 + 事务** 组合                   |

##### 七、一句话回答

> **"MultiLock 不就是像事务一样吗？那我为啥不用事务呢？"**

- **像的只是"全有或全无"的提交语义**，但 MultiLock 保证的是**多把锁的互斥获取**，事务保证的是**多步数据修改的原子提交/回滚**，**目标不同**。
- **事务不能代替 MultiLock**：事务不保证互斥，并发下照样超卖；跨库/跨服务事务也管不了。
- **MultiLock 不能代替事务**：它只管"谁持有锁"，不管"数据改没改对、失败回不回滚"。
- **正确做法是组合**：**MultiLock/分布式锁保证互斥，事务保证原子性**，各司其职，而不是互相替代。

**一句话：MultiLock 是"分布式互斥锁的组合"，事务是"数据原子性的保证"；前者管并发谁先动，后者管数据改对改全，像但不同，不能替代，只能配合。**



### Redisson总结

trylock的四个重载方法:

~~~java
// ① 无参：不等待，试一次（你贴的这个）
boolean tryLock();

// ② 只带 waitTime：在 waitTime 内重试，看门狗
boolean tryLock(long waitTime, TimeUnit unit);

// ③ 带 waitTime + leaseTime：在 waitTime 内重试，固定租期
boolean tryLock(long waitTime, long leaseTime, TimeUnit unit);   // ← 重试主体在这里

// ④ 带 leaseTime 的 lock：无限等 + 固定租期
void lock(long leaseTime, TimeUnit unit);
~~~



总结一下四个问题的核心解法：

| 问题         | 解决思路                      | 关键源码                                                     | 核心技术                            |
| :----------- | :---------------------------- | :----------------------------------------------------------- | :---------------------------------- |
| **可重入**   | String→Hash，value 存重入计数 | `tryLockInnerAsync`（Lua: exists→hset / hexists→hincrby+1）  | `HSET` + `HINCRBY` + Lua            |
| **可重试**   | Pub/Sub 事件驱动 + 信号量阻塞 | `tryLock` → `subscribe` → `semaphore.tryAcquire` → `onMessage` → `release` | `SUBSCRIBE`/`PUBLISH` + `Semaphore` |
| **超时释放** | 看门狗定期续命，不设死过期    | `scheduleExpirationRenewal` → `renewExpiration`（每 10s 跑 Lua: hexists→pexpire） | Netty `schedule` + Lua              |
| **主从一致** | 多独立节点过半表决            | `RedissonMultiLock.tryLock`（统计全部成功） / `RedissonRedLock`（过半判定） | `RedissonRedLock`                   |

四个解法的核心逻辑都是你熟悉的 Redis 命令（SETNX、Lua、EXPIRE、PUBLISH、HINCRBY），Redisson 的价值在于把这些命令组合成了工业级的工程封装。

### Redison的定位（学完后补）

==Redison就是借助redis实现的工具，用于解决可能出现并发场景的锁，redis本身是单线程的，没有并发问题==

> redison做的是分布式锁，但是我学习后面的秒杀优化时，其实用到的还是redis原生的命令。比如判断一人一单，可用redisson提供的锁加数据库操作来判断一人一单，也就是说这个锁用处就是用来处理比如涉及数据库并发的场景。但假如我判断一人一单的操作只在redis中进行了，其实还是要用到redis原生的命令，没办法在redis中用redisson吧？或者说对于redis本身就是单线程，不会有并发问题？
> 总结来说，redison就像是借助redis实现的高可用的锁，它的使用对象是哪些会有并发场景的业务，对于redis本身是不需要的？因为redis本身是线程安全的？

#### 一、Redis 本身为什么不需要锁

Redis 的**命令执行是单线程的**（6.0 引入的多线程只用于网络 IO 读写，命令执行仍是单线程），所以：

- 单条命令天然原子：`DECR stock` 不存在“扣到一半被别人插进来”的问题
- 一个 Lua 脚本也是原子的：Redis 执行脚本期间不会插入执行其他命令

所以在 **Redis 内部**，不存在并发修改问题，不需要锁。

#### 二、但有个关键细节：多条命令组合不原子

单条命令原子，**多条命令分开执行就不原子了**：

plaintext

```
// 这两步之间可能插入别人的命令
if (get("stock") > 0)        // 第1步：判断
    // ← 此瞬间别的线程可能把 stock 扣到 0
    decr("stock")            // 第2步：扣减 → 超卖
```

解法不是加锁，而是**用 Lua 脚本把多步压成一个原子操作**：

lua

```lua
-- 判断 + 扣减 压成一个脚本，整体原子
if tonumber(redis.call('get', KEYS[1])) > 0 then
    return redis.call('decr', KEYS[1])
else
    return -1
end
```

这就是秒杀优化里的做法：**能用 Lua 压缩的临界区，就不需要锁**。

#### 三、Redisson 锁的保护对象：Redis 外部的临界区

锁真正防的是“**判断和写入跨了两个系统**”的场景：

plaintext

```
// 判断在数据库，写入也在数据库 —— Redis 帮不上忙
orderId = queryDB("select * from order where user_id=? and voucher_id=?")
if (orderId == null)
    insertDB(order)      // ← 查和插之间，别的线程可能已经插了 → 重复下单
```

这段代码 Redis 管不了——它不是 Redis 命令，是数据库操作。这时才需要 Redisson 锁把这段包起来。

#### 四、对应到秒杀优化的两条路径

| 方案   | 一人一单判断在哪                               | 需要锁吗              | 靠什么保证原子          |
| :----- | :--------------------------------------------- | :-------------------- | :---------------------- |
| 基础版 | 数据库（查 + 插）                              | **要**（Redisson 锁） | 锁住"查库+插库"这段代码 |
| 优化版 | Redis（Lua 脚本判断库存+用户 Set 判重 + 扣减） | **不要**              | Lua 脚本整体原子        |

优化版的完整链路：Lua 脚本在 Redis 里原子完成“判库存 + 判一人一单 + 预扣减”→ 通过 MQ 异步发消息 → 消费者再写数据库。数据库写慢没关系，因为**准入资格已经在 Redis 里原子地控制住了**。

一句话总结

- **Redis**：单线程命令执行，自身线程安全，不需要锁；多步操作用 Lua 压成原子即可
- **Redisson**：借 Redis 实现的锁，保护对象是 **Redis 之外的临界区**（数据库操作、跨系统操作）
- 判断标准就一条：**临界区能不能压进一个 Lua 脚本？能 → 不用锁；不能（涉及数据库等外部系统）→ 用锁**

### 可重试-自旋和redison解决可重试的区别以及优缺点（学完后补）

![自旋与Redisson选型决策图](C:\Users\沨烨\Desktop\WorkBuddy\自旋与Redisson选型决策图.png)

#### 一、一句话本质

> **自旋 = pull（拉）**，成本压在**等待者**身上（CPU 空转 + Redis 轮询请求）。
> **Redisson = push（推）**​，成本转移到 **pub/sub 通道 + 唤醒机制**上。
> 两者回答的是同一个问题——「锁被别人占着时怎么等」，区别只是把成本放在了系统里不同的位置。

#### 二、全维度对比

| 维度         | 自旋轮询                                         | Redisson（Pub/Sub）                      |
| :----------- | :----------------------------------------------- | :--------------------------------------- |
| 等待模型     | 忙等轮询（pull）                                 | 发布订阅通知（push）                     |
| 谁发起重试   | 等待者反复去问                                   | 释放者 `publish`，等待者被唤醒           |
| 线程状态     | RUNNABLE，反复 sleep / 醒来                      | `LockSupport.park` 挂起                  |
| CPU 占用     | 持续空转                                         | **0**                                    |
| Redis 请求量 | 每次重试 1 个 `SETNX`，随「并发 × 频率」线性放大 | 订阅 1 次，等待期间 **0**                |
| 抢占延迟     | 平均半个轮询周期                                 | **近似实时**                             |
| 等待上限     | 通常没有 → 可能无限循环                          | `tryLock(waitTime, ...)` 可控            |
| 公平性       | 完全随机抢，可能饥饿                             | 默认也不公平（惊群），`FairLock` 才 FIFO |
| 可中断       | 差                                               | 好（`lockInterruptibly` 能打断 park）    |
| 可重入       | 不支持                                           | 支持（Hash `{UUID:threadId → count}`）   |
| 自动续期     | 无                                               | 看门狗（**仅无参 `lock()` 启用**）       |
| 实现复杂度   | 极低，5 行                                       | 需引依赖 + 理解三件套                    |
| 故障影响面   | 只影响等待者自己                                 | Redis / 通道异常时影响面更大             |

#### 三、优缺点速览

**自旋**

- ✅ 零依赖、全透明，每一步都看得见
- ✅ **临界区极短时反而更快**——省掉了订阅、建通道、唤醒三步开销
- ❌ CPU 空转；Redis 请求量随并发线性放大（100 线程 × 每 50ms = 2000 QPS 纯浪费）
- ❌ 无等待上限、无退避就是惊群、不支持可重入
- ❌ **自旋越猛，误删越容易触发**——锁一过期立刻被接手，旧持有者的 `DEL` 删的几乎必然是别人的锁

**Redisson**

- ✅ 等待期间 CPU 0、Redis 0 请求，资源占用与等待时长**无关**
- ✅ 释放即通知，延迟低且稳定
- ✅ 三件套齐全：可重入 + 自动续期 + 通知等待；API 完整（读写锁 / 信号量 / MultiLock）
- ❌ **惊群 + 非公平**：一次 `publish` 唤醒**全部**等待者，只有一个抢到，其余继续 park，顺序随机可能饥饿
- ❌ **配置坑**：`lock(leaseTime, unit)` 和 `tryLock(waitTime, leaseTime, unit)` **都会关掉看门狗**
- ❌ 主从切换可能丢锁（RedLock 是补丁，但 RedLock 本身争议极大）
- ❌ 黑盒，出问题排查成本高于自己写的几行代码

#### 四、最值得记住的一条

成熟实现**从不二选一**。`synchronized`、`ReentrantLock`、AQS 走的都是组合路线：

plaintext

```
先自旋几次（赌"锁马上释放"，省掉挂起/唤醒开销）
   ↓ 还拿不到
再 park 挂起（不赌了，把 CPU 让出去）
```

**纯自旋**和**纯 park** 都是两个极端——工程上的最优解是「短临界区自旋、长临界区阻塞」。这也说明 Redisson 不是"更好的自旋"，而是**直接跳过了自旋这个阶段**。

#### 五、选型速查

| 场景                                 | 方案                                |
| :----------------------------------- | :---------------------------------- |
| 临界区毫秒级 + 并发低 + 学习/演示    | 自旋，**必须**加随机退避 + 等待上限 |
| 临界区耗时不预判 / 长事务 / 要可重入 | Redisson `lock()` + 业务层超时      |
| 需要公平排队                         | `RedissonFairLock`                  |
| 需要响应中断                         | `lockInterruptibly()`               |
| 拿不到就返回，不等                   | `tryLock(0, leaseTime, unit)`       |

## Spring 依赖注入三种方式（补充知识）

![Spring依赖注入三种方式对比](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/Spring%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5%E4%B8%89%E7%A7%8D%E6%96%B9%E5%BC%8F%E5%AF%B9%E6%AF%94.png)

### 一、三种注入方式

#### 1. 字段注入（你项目里现在用的）

```java
@Service
public class VoucherOrderServiceImpl {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;  // Spring 通过反射直接写进去
}
```

Spring 实例化 Bean 之后，通过**反射**找到标了 `@Autowired` 的字段，从容器里找到对应类型的 Bean，用 `field.set(bean, dependency)` 写入。

**问题：**

- 字段不能声明 `final` — 对象创建后依赖还没进来，存在"半初始化"窗口
- 单元测试时没法 `new` 出来传 mock — 必须依赖 Spring 容器或反射工具
- 循环依赖被 Spring 偷偷解决了（通过三级缓存），你感知不到，但这是个隐患
- 依赖多了你也不觉得 — 一个类塞 20 个 `@Autowired` 字段也不报错，违反单一职责

#### 2. 构造器注入（Spring 官方推荐）

```java
@Service
public class VoucherOrderServiceImpl {
    private final StringRedisTemplate stringRedisTemplate;

    // 只有一个构造器时 @Autowired 可省略
    public VoucherOrderServiceImpl(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }
}
```

Spring 在**实例化阶段**就调用构造器，把依赖作为参数传入。对象一出生就是完整的。

**关键区别：** 字段注入是"先创建空对象，再反射填字段"；构造器注入是"创建对象时就传入依赖"。这决定了对象能不能声明 `final`。

**优势：**

- `final` 字段 — 创建后不可变，线程安全
- 单元测试直接 `new VoucherOrderServiceImpl(mockTemplate)` — 不需要 Spring
- 循环依赖**启动直接报错** — Spring 发现 A 的构造器需要 B，B 的构造器又需要 A，谁也创建不出来，立刻暴露问题
- 依赖太多时构造器参数会很长 — 这本身就是一种"该拆类了"的信号

==**一个细节：** 如果一个类只有**一个构造器**，Spring 4.3+ 会自动用它注入，不需要写 `@Autowired`。多个构造器时才需要指定哪个加 `@Autowired`。==

#### 3. Setter 注入

```java
@Service
public class SomeService {
    private OptionalDep dep;

    @Autowired
    public void setDep(OptionalDep dep) {
        this.dep = dep;
    }
}
```

对象先通过无参构造器创建，再调 setter 注入。适合**可选依赖**（可以不注入，对象照样能用）。现在很少用。

### 二、Spring 是怎么找到要注入的 Bean 的？

这涉及到**注入注解**的体系，有三个来源：

| 注解         | 来源         | 匹配策略                       | 附带能力                  |
| :----------- | :----------- | :----------------------------- | :------------------------ |
| `@Autowired` | Spring 自带  | **先 byType**，多个时再 byName | `required=false` 可选注入 |
| `@Resource`  | JSR-250 标准 | **先 byName**，找不到再 byType | 可指定 `name` 和 `type`   |
| `@Inject`    | JSR-330 标准 | byType                         | 需要额外依赖              |

#### byType 匹配冲突怎么办？

当容器里有多个同类型 Bean 时（比如两个 `DataSource`），`@Autowired` byType 会报错。解决方案：

```java
// 方案1：@Qualifier 指定名字
@Autowired
@Qualifier("readDataSource")
private DataSource dataSource;

// 方案2：字段名当byName的key（字段名 = Bean名）
@Autowired
private DataSource readDataSource;  // Spring 会用 "readDataSource" 去匹配

// 方案3：某个Bean标 @Primary，byType时优先选它
@Bean
@Primary
public DataSource readDataSource() { ... }
```

#### @Resource 的独特之处

```java
@Resource                          // 先按字段名 "seckillVoucherService" 找
@Resource(name = "xxx")            // 直接按指定name找
@Resource(type = SeckillVoucherServiceImpl.class)  // 按type找
```

你项目里大量用的 `@Resource` 走的是**先 byName** 逻辑 — 字段名就是 Bean 名（类名首字母小写）。这也是为什么很多教程用 `@Resource`：只要字段名写对，基本不会冲突。

### 三、Bean 生命周期中的注入时机

再看一张图，展示注入在 Bean 生命周期中的位置。

![image-20260808210737443](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260808210737443.png)

上图展示了 Bean 生命周期中三种注入的时机差异。下面补充几个关键知识点。

### 四、注入的前提：Bean 必须被 Spring 管理

这是你上一个问题的核心。注入（无论哪种方式）的前提是：**目标对象是 Spring 容器创建和管理的 Bean**。

Spring 管理 Bean 的方式有两种：

#### 1. 组件扫描（Component Scanning）

```java
@Component        // 通用
@Service          // 业务层
@Repository       // 持久层
@Controller       // 表现层
@Configuration    // 配置类
```

Spring 启动时扫描这些注解标记的类，自动注册为 Bean。你的 `VoucherOrderServiceImpl` 标了 `@Service`，所以 Spring 管它。

#### 2. @Bean 方法

```java
@Configuration
public class RedisConfig {
    @Bean
    public StringRedisTemplate stringRedisTemplate() {
        return new StringRedisTemplate();
    }
}
```

方法返回值注册为 Bean，方法名就是 Bean 名。

### 五、循环依赖（字段注入能活、构造器注入必死）（不懂，留个扣子）

......

### 六、总结一张表

| 维度         | 字段注入                      | 构造器注入                | Setter注入   |
| :----------- | :---------------------------- | :------------------------ | :----------- |
| 注解位置     | 字段上                        | 构造器上(可省略)          | setter方法上 |
| final字段    | 不可以                        | **可以**                  | 不可以       |
| 注入时机     | 实例化之后                    | 实例化时                  | 实例化之后   |
| 循环依赖     | 三级缓存解决                  | **直接报错**              | 三级缓存解决 |
| 单元测试     | 困难(需反射)                  | **直接new**               | 可new+set    |
| Spring推荐   | 不推荐                        | **推荐**                  | 可选依赖时用 |
| 你项目的用法 | `@Autowired`/`@Resource` 字段 | SimpleRedisLock(手动传参) | 未使用       |

实际项目中，**强依赖用构造器注入，可选依赖用Setter**，字段注入能少用就少用。不过学习项目里图方便用 `@Autowired`/`@Resource` 字段注入也完全没问题，知道区别就行。

## 项目中涉及Spring依赖注入的疑问

> 实战篇11SimpleRedisLock构造器传入了stringRedisTemplate，为啥VoucherOrderServiceImpl使用他时还是手动传入了stringRedisTemplate？

==这个问题涉及 《Spring 依赖注入三种方式（补充知识）》四、注入的前提：Bean 必须被 Spring 管理==

SimpleRedisLock使用构造器传入stringRedisTemplate，这个并不是构造器注入（类上没有标记注解被spring管理），只是单纯的定义传入stringRedisTemplate的逻辑。

**`SimpleRedisLock` 两条路都没走** — 没注解，没有 `@Bean` 方法。所以 Spring 不认识它，`new` 出来就是普通 Java 对象，构造器里的参数 Spring 不会帮你填。

>  如果我用构造器注入的话，那我new `SimpleRedisLock`时还要传stringRedisTemplate进去吗？

要分两种情况说，关键在于**谁创建这个对象**。

<h4>情况一：SimpleRedisLock 不加 @Component（当前状态）</h4>

你 `new` 它，你就是创建者。构造器写什么参数，你就得传什么：

```java
// 你 new 的，你传参，Spring 不管
SimpleRedisLock lock = new SimpleRedisLock("order:"+userid, stringRedisTemplate);
```

这里不存在"注入"这件事。构造器注入是 **Spring 调构造器时**才叫注入，你手动 `new` 调构造器叫**传参**，两码事。

<h4>情况二：SimpleRedisLock 加 @Component</h4>

Spring 成为创建者，Spring 调构造器时自动从容器里找 `StringRedisTemplate` 传进去。你**不再 `new`**，而是让 Spring 注入：

```java
@Component
public class SimpleRedisLock implements ILock {
    private final StringRedisTemplate stringRedisTemplate;
    private final String name;

    // Spring 调这个构造器，自动传 stringRedisTemplate
    public SimpleRedisLock(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }
}
```

但问题来了 — `name` 每次不同（锁不同用户），不能写死在构造器里。所以得改成方法参数：

```java
@Component
public class SimpleRedisLock implements ILock {
    private final StringRedisTemplate stringRedisTemplate;

    public SimpleRedisLock(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;  // Spring自动注入
    }

    // name 作为方法参数，每次调用时传
    public boolean tryLock(String name, long timeoutSec) { ... }
    public void unLock(String name) { ... }
}
```

然后 VoucherOrderServiceImpl 里直接注入这个 Bean，不再 `new`：

```java
@Service
public class VoucherOrderServiceImpl {
    @Autowired
    private SimpleRedisLock simpleRedisLock;  // Spring 注入，不用你 new

    public Result sekillVoucher(Long voucherId) {
        // ...
        boolean isLock = simpleRedisLock.tryLock("order:" + userid, 1200);
        // ...
        simpleRedisLock.unLock("order:" + userid);
    }
}
```

<h4>核心结论</h4>

| 谁创建对象                         | 谁负责传依赖  | 你要做的                              |
| :--------------------------------- | :------------ | :------------------------------------ |
| **你 `new`**                       | 你自己传      | 手动传参，构造器注入不生效            |
| **Spring 创建**（标了 @Component） | Spring 自动传 | 不用 `new`，直接 `@Autowired` 拿 Bean |

**一句话：构造器注入 = Spring 替你调构造器并传参。你一旦自己 `new`，Spring 就不参与了，参数全得你自己传。**

所以当前项目的做法完全合理 — SimpleRedisLock 是个每次 `new` 的短生命周期对象，`name` 每次不同，做成 Spring Bean 反而别扭。`stringRedisTemplate` 从 Service 里搬过来传给它，就是最简单直接的方式。

## 死锁的四种情况（补充知识，学到Redission拓展而来）（先记录，迟早要看！！！）

![死锁的四种主要类型](C:\Users\沨烨\Desktop\WorkBuddy\死锁的四种主要类型.png)

下面逐个展开讲，配代码示例。

### 1. 经典互斥死锁（最常见）

**本质：多个线程以不同顺序获取多把锁，形成循环等待。**

```java
// 线程A：先锁1后锁2
new Thread(() -> {
    synchronized(lock1) {
        synchronized(lock2) { ... }
    }
}).start();

// 线程B：先锁2后锁1
new Thread(() -> {
    synchronized(lock2) {
        synchronized(lock1) { ... }
    }
}).start();
```

A 拿了锁1等锁2，B 拿了锁2等锁1 → 永久卡死。

**怎么避免：** 所有线程按**固定顺序**加锁。不管 A 还是 B，都先锁1再锁2，就不会循环等待。

### 2. 自死锁（重入死锁）

**本质：同一线程对同一把不可重入锁请求两次，等自己释放。**

```java
// 不可重入锁
public void methodA() {
    lock.lock();      // 线程A拿到锁
    methodB();        // → methodB也尝试拿锁
    lock.unlock();
}

public void methodB() {
    lock.lock();      // 锁已被自己持有，阻塞等待自己释放 → 卡死
    lock.unlock();
}
```

**和经典死锁的区别：** 只有 1 个线程 1 把锁，线程等的是自己。经典死锁是 2 个线程 2 把锁互相等。

**怎么避免：** 用可重入锁（`ReentrantLock`、`synchronized` 天然可重入），或者重构代码避免嵌套加锁。

### 3. 数据库死锁

**本质：和经典死锁原理一样，但锁的粒度是数据库行，不是代码里的锁对象。**

```sql
-- 事务A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁住行1
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等行2的锁

-- 事务B（同时）
BEGIN;
UPDATE account SET balance = balance - 50 WHERE id = 2;   -- 锁住行2
UPDATE account SET balance = balance + 50 WHERE id = 1;   -- 等行1的锁
```

A 锁行1等行2，B 锁行2等行1 → 死锁。

**和代码死锁的区别：**

- MySQL InnoDB 有**自动死锁检测** — 发现死锁后主动回滚代价较小的事务，抛 `DeadlockLoserDataAccessException`
- 代码层面的 `synchronized` 死锁 JVM **不检测**，直接永久卡死
- 你项目里 `@Transactional` + 并发更新同一批数据就可能遇到

**怎么避免：** 批量更新时按主键排序后统一加锁顺序。

### 4. 分布式死锁

**本质：经典死锁的分布式版本，锁存在 Redis 里而不是 JVM 里。**

```
节点A: 获取Redis锁lock:1 → 成功 → 获取lock:2 → 等待
节点B: 获取Redis锁lock:2 → 成功 → 获取lock:1 → 等待
```

**和单机死锁的区别：**

- 单机死锁 JVM 能看到所有线程状态，但 `synchronized` 不检测（`jstack` 能看到）
- 分布式死锁跨 JVM，**没有全局视角**，`jstack` 看不到对面节点在干什么
- 只能靠锁超时兜底 — 但如果业务执行时间超过超时时间，又会引发误删问题（你之前学过的那个）

**你的项目目前只有 1 把锁（`lock:order:userid`），不会出现分布式死锁。** 除非同时用多把不同 key 的分布式锁且加锁顺序不一致。

### 死锁的四个必要条件（Coffman 条件）

死锁发生必须**同时满足**这四个条件，打破任意一个就能避免：

| 条件       | 含义                           | 打破方式                        |
| :--------- | :----------------------------- | :------------------------------ |
| 互斥       | 资源同一时刻只能被一个线程持有 | 有些场景可以用读写锁（共享读）  |
| 持有并等待 | 拿着锁1不放，去等锁2           | 一次性获取所有锁，拿不到就不拿  |
| 不可剥夺   | 不能强行抢别人的锁             | `tryLock(timeout)` 超时自动放弃 |
| 循环等待   | A等B，B等A，形成环             | 固定加锁顺序，破环              |

**最实用的是后两个：** `tryLock` 带超时（打破不可剥夺）和固定加锁顺序（打破循环等待）。

### 两个容易混淆的概念（不是死锁）

**活锁（Livelock）：** 线程没卡住，一直在跑，但就是做不下去。比如两个线程互相谦让，你让我我先让你，结果谁也不往前走。和死锁的区别是线程是**活的**，但和死了一样没用。

**饥饿（Starvation）：** 线程一直拿不到锁，不是因为死锁，而是总被别的线程插队。比如用 `synchronized` 是非公平锁，新来的线程可能比等了很久的线程先拿到锁，老的线程就一直饿着。`ReentrantLock(true)` 公平锁能解决这个问题。

## 秒杀优化

### 实战篇23（基于redis完成库存余量和一人一单判断--lua脚本）

> lua脚本中`if(redis.call('sismember', orderKey, userId) == 1) `最开始的个订单key没有事先存进去，还能判断出来吗？

<h4>Redis 对不存在 key 的处理规则</h4>

Redis 有一个核心设计原则：**对不存在的 key 进行读操作时，把它当作对应类型的"空容器"处理，返回空结果，但不会创建 key**。

具体到你这段脚本：

| 命令        | key 不存在时的行为            | 会创建 key？ |
| :---------- | :---------------------------- | :----------- |
| `SISMEMBER` | 返回 `0`（member 不在集合中） | ❌ 不会       |
| `GET`       | 返回 `nil`                    | ❌ 不会       |
| `SADD`      | 自动创建 set 并添加成员       | ✅ 会         |
| `SET`       | 自动创建 string               | ✅ 会         |

<h4>你的脚本执行流程（第一次下单）</h4>

```
orderKey = 'seckill:order:123'  ← 此时不存在

1. SISMEMBER orderKey userId  →  返回 0
   Redis 把不存在的 key 当空集合，自然不是成员，返回 0
   条件不成立，不 return 2，继续往下走

2. INCRBY stockKey -1  →  扣库存

3. SADD orderKey userId  →  这一步才真正创建 orderKey
   Redis 自动创建 set，把 userId 加进去

4. XADD stream.orders ...  →  发消息
```

第二次同一个用户来下单时，`orderKey` 已经存在了，`SISMEMBER` 返回 `1`，命中 `return 2`，拦截重复下单。

<h4>为什么这样设计很合理</h4>

这其实是 Redis 的**懒创建**模式，好处是：

1. **不用预热数据**——不用提前为每个优惠券创建空 set
2. **节省内存**——没下过单的优惠券根本不占 set 的内存
3. **逻辑正确**——空集合的 `SISMEMBER` 返回 0，语义上就是"这个用户没下过单"，完全符合业务预期

> `if(tonumber(redis.call('get', stockKey)) <= 0) `为啥这里要tonumber呢，为啥`if(redis.call('sismember', orderKey, userId) == 1)`就不用呢？

<h4>第一层：存储层（所有 value 确实都是字符串）</h4>

Redis 没有独立的整数类型，所有 value（包括 `INCR` 维护的数字）底层都是以**字符串**形式存储的。这一点没问题。

<h4>第二层：协议回复层（决定要不要 tonumber 的关键）</h4>

`redis.call` 的返回值**不是统一为字符串**，而是由该命令的**回复类型**决定，Redis 会自动转换成对应的 Lua 类型：

| Redis 回复类型        | 典型命令                             | 转成 Lua 类型 | 需要 tonumber？ |
| :-------------------- | :----------------------------------- | :------------ | :-------------- |
| **Integer reply**     | `SISMEMBER`, `EXISTS`, `DEL`, `INCR` | `number`      | ❌ 不需要        |
| **Bulk string reply** | `GET`, `HGET`, `GETRANGE`            | `string`      | ✅ 需要          |
| **Status reply**      | `SET` 返回 OK                        | `string`      | ✅ 需要          |
| **Nil reply**         | `GET` 不存在的 key                   | `false`       | —               |

所以：

```lua
-- GET 返回 Bulk string → Lua 是 string："10"
redis.call('get', stockKey)   -- "10"

-- Lua 强类型，string 和 number 比较直接报错
"10" <= 0   -- ❌ attempt to compare string with number
-- 所以必须：
tonumber("10") <= 0   -- ✅ 10 <= 0
```

```lua
-- SISMEMBER 返回 Integer → Lua 是 number：0 或 1
redis.call('sismember', orderKey, userId)  -- 0 或 1

-- 类型本身是 number，直接比较没问题
1 == 1   -- ✅
```

<h4>一句话总结</h4>

**决定要不要 `tonumber` 的不是"存储层是不是字符串"，而是"这个命令的回复类型是什么"**：计数/判断类命令（SISMEMBER、EXISTS）返回整数 → 不需要；取值类命令（GET、HGET）返回字符串 → 需要。

> `local orderKey = 'seckill:order:' .. voucherId`这里为什么不连接订单id做key呢？(仔细看看，虽然定义是订单key，但并没有用订单id做拼接)

这其实是个**数据模型设计**的问题。关键要理解 `orderKey` 的职责是什么。

<h4>`orderKey` 的职责：记录"谁买过这个券"</h4>

```lua
-- 这行存的是：一个 set，成员是 userId
SADD orderKey userId   -- seckill:order:1001 → { "user_1", "user_2", ... }
```

它的目的是回答一个问题：**"用户 X 是不是已经买过优惠券 Y 了？"**

- 查询维度是 **券**（一个券对应一个集合）
- 集合里存的是 **用户 id**（多个用户）
- 这是一对多的关系：`1 个优惠券 → N 个已购用户`

所以 key 必须按 **voucherId** 维度组织，成员存 **userId**，数据模型才是对的。

<h4>如果用 orderId 做 key 会怎样？</h4>

orderId 是**每次下单唯一生成**的，如果用 `seckill:order:{orderId}` 做 key：

```
第一次下单 → seckill:order:10001 → { user_1 }
第二次下单 → seckill:order:10002 → { user_1 }   ← 又是新 key
```

每笔订单一个 key，每个 key 里只有一条记录。那么：

```lua
SISMEMBER orderKey userId  -- 无论怎么查都是 0
```

==**重复下单检测直接失效**——因为每个 key 只对应一笔订单，永远查不到"该用户以前买过"。==

### 为什么提前提取代理对象为成员变量，异步线程才能成功执行到事务方法

~~~java
try {
	//注意：由于是spring的事务是放在threadLocal中，此时的是多线程，事务会失效
        proxy.createVoucherOrder(voucherOrder);
}
~~~

==首先理解一下代理对象==

### 1、代理对象是什么，什么情况下会生成代理对象？

#### **代理对象是什么**：

一个和原始对象实现相同接口的"替身"。

代理对象是 Spring 在**容器启动时**生成的，它实现了和原始对象相同的接口，内部持有两样东西：

- **target**（原始对象）— 你写的 `VoucherOrderServiceImpl` 实例，你的业务代码都在这里
- **方法→拦截器链 映射表** — 记录每个方法要不要走拦截器、走哪些拦截器

==外部==调任何方法都先进代理对象，代理对象查映射表决定加不加料，再转发给 target 执行。

~~~java
// 你写的类（原始对象）
@Service
public class VoucherOrderServiceImpl implements IVoucherOrderService {
    @Transactional
    public void createVoucherOrder1(VoucherOrder voucherOrder) {
        // 你的业务代码
    }
}

// Spring 运行时实际生成的（概念上的结构，不是真实代码）
class VoucherOrderServiceImpl$Proxy implements IVoucherOrderService {
    private VoucherOrderServiceImpl target;    // 持有原始对象的引用
    private List<Advice> interceptors;        // 拦截器链（事务、日志等）

    public void createVoucherOrder1(VoucherOrder order) {
        // 调用前：执行拦截器链
        //  → 开启事务（@Transactional通知）
        //  → 设置ThreadLocal（ExposeInvocationInterceptor）
        
        target.createVoucherOrder1(order);    // 调用原始对象的方法
        
        // 调用后：执行拦截器链
        //  → 提交/回滚事务
        //  → 清除ThreadLocal
    }
}

~~~

> 代理对象复制了原始对象的**所有** public 方法，不管有没有 `@Transactional`。但只有匹配切面的方法在调用时才执行拦截器链，不匹配的直接透传给原始对象。

![Spring_AOP代理对象工作机制](C:\Users\沨烨\Desktop\WorkBuddy\Spring_AOP代理对象工作机制.png)

==也就是说调用方法可能走三条路（前提是有代理对象生成）==

1. 走内部原始的方法 `this.`调用
2. 走外部代理对象中没有匹配切面的方法，此时不会额外的调用拦截器链，直接用原始对象的引用来运行方法
3. 走代理对象匹配切面的方法（如方法上标注了`@Transactional`）,先走事务拦截器，再使用原始对象引用运行方法



#### **生成条件**：

Spring 容器启动时，`AnnotationAwareAspectJAutoProxyCreator`（一个 BeanPostProcessor）在 Bean 初始化阶段扫描每个 Bean 的方法。只要类里有**至少一个方法**匹配了某个切面（比如标了 `@Transactional`），Spring 就会给**整个类**生成代理对象。一个 `@Transactional` 方法就够触发，但代理会包裹类的所有 public 方法。

### 2. 代理对象存在于哪里？是特定的某个线程吗？

**存在于 JVM 堆内存，不属于任何线程。** 代理对象是单例，和普通 Java 对象一样存在堆里，所有线程通过引用访问的是同一个对象。线程不"拥有"代理对象，线程只是持有它的引用去调用方法。

### 3.生成代理对象后，什么情况下会走拦截器（开始事务）？

在1中有提到调用方法走的三条路中的第三条

#### 事务拦截器什么时候工作

**两个条件缺一不可：**

1. **方法匹配了事务切面** — 方法上标了 `@Transactional`，Spring 在启动时就把这个方法加进了拦截器链映射表
2. **调用经过了代理对象** — 外部通过代理对象调用，代理对象才有机会查映射表、执行拦截器链

### 异步线程直接AopContext.currentProxy()依然执行不了事务的原因

#### 1.AopContext.currentProxy()是获取代理对象，不是生成代理对象

`AopContext.currentProxy()` 获取代理对象的机制是**从当前线程的 ThreadLocal 里读**，不是从堆里"找"代理对象。

~~~java
private static final ThreadLocal<Object> currentProxy = new NamedThreadLocal("Current AOP proxy");
~~~

#### 2. ThreadLocal 什么时候被塞入代理引用？

发生在**「代理对象的方法被调用」的那一刻**。具体到 Spring 源码，在 `JdkDynamicAopProxy.invoke()`（CGLIB 代理同理）的开头，如果 `exposeProxy = true`，会执行：

```java
// 进入代理方法
oldProxy = AopContext.setCurrentProxy(proxy);  // ← 塞入当前线程的 ThreadLocal
setProxyContext = true;
try {
    // 跑拦截器链 → 事务拦截器开启事务 → 调用目标方法
    return method.invoke(target);
} finally {
    if (setProxyContext) {
        AopContext.setCurrentProxy(oldProxy);   // ← 退出时恢复（清理）
    }
}
```

所以结论是：**每次经过代理对象的调用，进入时 set、退出时 clear，周而复始。** `AopContext.currentProxy()` 只是一个读取器——它从 ThreadLocal 里读，本身不写入任何东西。

#### 3.==那为什么在异步线程里用 `currentProxy()` 拿不到代理？==

首先给个结论，异步线程在初次使用`AopContext.currentProxy()`是拿自己的`ThreadLocal`中的代理对象取到的结果是`null`。也就是说异步线程自己的ThreadLocal中没有代理对象的引用。

这里有个死循环：想要从ThreadLocal中拿代理对象--》代理对象得先被调用--》代理对象想被调用--》得从ThreadLocal中拿到对象

![image-20260815214122160](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260815214122160.png)

==主线程有框架帮它打破了这个调用循环，而异步线程中没有==

#### 关键点：

主线程的`ThreadLocal`中`Layer 1: Spring DispatcherServlet（框架层）`不通过`ThreadLocal`直接就能拿到并反射代理对象，由于执行了代理对象，代理对象的引用同时被放进了`ThreadLocal`,此时主线程用`AopContext.currentProxy()`自然能取到代理对象。

异步线程中没有人提前将代理引用放进它自己的`ThreadLocal`中，使用这种`AopContext.currentProxy()`方法自然取自己线程`ThreadLocal`的代理对象时为`null`，然后抛会抛空指针异常。

![主线程调用代理对象过程](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260815212956652.png)

==两者区别：==

![主线程和异步线程区别](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20260815214601029.png)

## Redis消息队列

### 实战篇29

> 创建消费者组要id参数有啥用呢?

![image-20260816155000710](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260816155000710.png)

这个 ID 参数不仅存在，而且**极其重要**，它决定了消费者组**从哪个位置开始消费**。

我来给你讲透它的三个核心作用：

<h4>1. 设定消费的“初始起点”</h4>

消费者组创建后，组内成员（消费者）第一次去读取消息时，到底从哪条消息开始读？**就是由这个 ID 决定的**。

它定义了消费者组的**初始游标（last_delivered_id）**。

<h4>2. `ID` 的两个最经典传参（和你理解的一致）</h4>

虽然你的理解基本正确，但需要稍微精确一点：

- **传 `0`**：表示**从头开始**。消费者组会读取 Stream 中**所有已存在的历史消息**。一般用于数据迁移、重新处理历史数据或初始化场景。
- **传 `$`**：表示**只读未来的消息**。消费者组会**跳过**当前 Stream 中已经存在的所有消息，只读取**创建消费者组之后**才进来的新消息。一般用于新上线的业务，只想处理未来的增量数据。



------

**补充一点**：除了 `0` 和 `$`，你还可以传一个**具体的消息 ID**（比如 `1786863379555-0`），这样消费者组就会从这个 ID **之后**的消息开始读取，非常灵活。

------



<h4>3. 维护消费者组的“进度状态”（最关键）</h4>

这是 ID 参数最底层的用处。Redis 内部会为每个消费者组记录一个 **`last_delivered_id`**。

- 当你用 `XREADGROUP` 读取消息时，Redis 会自动更新这个 ID。
- 如果你把组里的某个消费者删了，或者组被重建了，**只要这个 ID 在，新的消费者就知道自己上次读到哪了**，**绝对不会重复消费**之前已经读过的消息。
- 这个 ID 参数，就是给这个“进度指针”设置一个**初始值**。

<h4>举一个最简单的对比例子</h4>

假设 Stream `users` 里已经有 5 条消息了。

- **场景 A：`XGROUP CREATE users g1 0`**
  - **效果**：组里第一个消费者去读时，会把这 5 条历史消息全读出来。
  - **用途**：以前没用过流，现在想把这 5 条消息也处理掉。
- **场景 B：`XGROUP CREATE users g2 $`**
  - **效果**：组里第一个消费者去读时，只会读到**第 6 条（未来的消息）**。之前那 5 条它看都不看。
  - **用途**：这个流之前已经被别的组处理过了，新组只关心以后新来的数据。

<h4>总结</h4>

这个 **ID 参数** 就是**给消费者组设定一个“初始的起跑线”**。它决定了你是要“补全历史记录”还是“只关注未来”。

所以，这个操作是绝对存在的，而且如果你的消费者组**不加这个参数**（比如省略掉），Redis 就没法知道你到底想从哪开始，也就没法创建组了。

> 这不是XREADGROUP干的事吗？`>`从下一个未消费的开始读 其他标识代表从以消费未确认的消息开始读？

你的理解方向是对的，但需要稍微区分一下**“创建时的设定”**和**“读取时的行为”**。我们来把这两件事拆开看：

<h4>1. 你提到的“从下一个未消费的开始读” —— 这是 `XREADGROUP` 干的</h4>

**没错！** 当消费者组**已经创建好**之后，消费者执行 `XREADGROUP` 时，**不需要**再传具体的 ID，而是传特殊符号 **`>`**。

- **`>`**：告诉 Redis **“从消费者组记录的最新位置开始，读所有未被组内任何消费者读过的新消息”**。
- **其他 ID（比如 `0`）**：如果 `XREADGROUP` 里传的不是 `>` 而是 `0`，那意思就变成了：**“读所有已经投递给消费者、但还没被 `XACK` 确认的待处理消息（Pending Messages）”**。

**所以，消费时的逻辑，确实和你理解的一模一样：**

- 传 `>` → 读**未消费**的新消息。
- 传 `0` → 读**已消费但未确认**的待处理消息（用于故障恢复）。

<h4>2. 那么 `XGROUP CREATE` 里的 `ID` 是干什么的？（区别在这里）</h4>

`XGROUP CREATE` 里的 `ID`，并不是为了“读取”使用的，而是为了==**给消费者组定“起跑线”**==。

**打个比喻：**

- `XGROUP CREATE` 里的 `ID`：像是**给新建的马拉松队伍发一张“出发时刻表”**。你是选择从“第 0 米”（`0`）开始跑，还是从“当前终点线”（`$`）开始跑。
- `XREADGROUP` 里的 `>`：像是**枪声响起后，队伍真正开始往前跑的动作**。

**关键区别：**
`XGROUP CREATE` 设置好这个 `ID` 后，组里那个“进度指针”就固定在这个位置了。以后你执行 `XREADGROUP ... >` 时，就是**从那个起跑线开始，往下读**。

<h4>3. 非常具体的对比（帮你彻底分清）</h4>

| 命令                | 参数            | 目的         | 读取范围                                                     |
| :------------------ | :-------------- | :----------- | :----------------------------------------------------------- |
| **`XGROUP CREATE`** | `0`             | **定起点**   | 指定这个组将来从“Stream 第一条消息”开始接管。                |
| **`XGROUP CREATE`** | `$`             | **定起点**   | 指定这个组将来从“创建那一刻之后的新消息”开始接管。           |
| **`XREADGROUP`**    | `>`             | **读新消息** | 从该组**设定的起点**继续往前推进，读还没发过的消息。         |
| **`XREADGROUP`**    | `0`（或其他ID） | **读待处理** | 专门翻找**已经发给消费者、但还没 ACK 确认**的“半死不活”的消息。 |

<h4>结论</h4>

- 你说得对：**“从下一个未消费的开始读”**，确实是 `XREADGROUP` 里传 `>` 这个参数干的事。
- `XGROUP CREATE` 里的 ID，只是**提前把这个组的“起步位置”设定好**，方便后面 `XREADGROUP >` 去执行。

**简单一句话总结：** 一个是“设起点”，一个是“按起点去跑”。这两个参数是配套使用的，不是重复的。

### 实战篇30（java中实现stream消息队列）

> xreadgroup 命令不是有noxack吗？为啥不加上这个自动ack一下呢？出了异常，我手动ack没问题？有必要在读取阶段手动ack一下吗？

![image-20260816163505308](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260816163505308.png)

==因为读取后还要有`createVoucherOrder()`落库操作，这个过程有可能失败。这个间隙正是 PEL 要保护的。用了 NOACK，这层保护就没了。==

这里有个关键的概念混淆——**NOACK 不是"自动帮你 ack"，而是"读取时立即 ack，连 PEL 都不进"**。这恰恰会破坏你代码里的异常恢复机制。

<h4>先搞清两种模式的区别</h4>

<h5>当前代码（不用 NOACK，手动 ack）</h5>

```
XREADGROUP ... → 消息进入 PEL → 处理 → 成功后 XACK → 从 PEL 移除
                                    → 异常 → 消息留在 PEL → handlePendingList() 恢复
```

消息读出来后会**先进入 PEL（Pending Entry List）**，在你 ack 之前一直挂着。如果处理成功，ack 掉它；如果抛异常，消息还在 PEL 里没丢，`handlePendingList()` 能把它捞回来重新处理。

<h5>如果用 NOACK</h5>

```
XREADGROUP ... NOACK → 消息直接 ack，不进 PEL → 处理
                                              → 异常 → 消息已经不在 PEL 里了 → 彻底丢了
```

`handlePendingList()` 去查 PEL，发现什么都没有——因为消息压根没进过 PEL。**异常恢复机制直接失效。**

<h4>逐条回应你的三个疑问</h4>

**1. "为啥不加上 NOACK 自动 ack 一下呢？"**

因为 NOACK 的语义是"我读完就当处理完了"。但你的场景是"读完后还要 `createVoucherOrder` 落库"——读和处理之间有个可能失败的操作，这个间隙正是 PEL 要保护的。用了 NOACK，这层保护就没了。

**2. "就算出了异常，我也可以手动 ack 吧？"**

这里反了。用了 NOACK 之后，消息**在读取时就已经被 ack 了**，你不需要也无法再 ack——它在 Redis 那边已经标记为"已处理"。而且如果处理异常了，你应该做的是**不 ack**（让它留在 PEL 等恢复），而不是去 ack 它——ack 一个处理失败的消息等于告诉 Redis"这个我处理好了"，消息就真没了。

**3. "有必要在读取阶段手动 ack 一下吧？"**

注意，代码里的 `XACK` 不是在"读取阶段"，而是在**处理成功之后**：

```java
// 3.创建订单（如果这一步抛异常，ack 不会执行）
createVoucherOrder(voucherOrder);
// 4.确认消息 XACK（只有成功走到这里才 ack）
stringRedisTemplate.opsForStream().acknowledge("s1", "g1", record.getId());
```

这个顺序是刻意设计的：**先干活，干成了再确认。** 如果 `createVoucherOrder` 抛异常，ack 那行根本不会执行，消息留在 PEL 里，`catch` 里的 `handlePendingList()` 就能捞到它重试。

<h4>一句话总结</h4>

|                        | 不用 NOACK（当前代码） | 用 NOACK             |
| :--------------------- | :--------------------- | :------------------- |
| 消息进 PEL             | ✅ 进                   | ❌ 不进               |
| 处理失败可恢复         | ✅ 从 PEL 捞回重试      | ❌ 彻底丢失           |
| handlePendingList 有效 | ✅ 有                   | ❌ 空转，PEN 里没东西 |

**PEL 的存在就是为了"处理可能失败"这件事。** ==NOACK 适合的场景是"读出来就等于处理完了"（比如纯日志消费、数据同步到另一个只追加的存储），不需要重试保障==。你这个秒杀订单创建场景，一旦丢失就是真实业务损失，必须保留 PEL 兜底，所以不能用 NOACK。

> 讲解一下获取订单信息返回list这一段代码中都干了啥操作，标注每个操作代表啥意思

<h4>① `stringRedisTemplate.opsForStream()`</h4>

从 `StringRedisTemplate` 拿到 **Stream 操作入口对象**。相当于告诉 Redis：「我接下来要对 Stream 数据结构做操作」。

<h4>② `.read(...)`</h4>

执行读取操作，底层对应 Redis 命令 `XREADGROUP`。

<h4>③ `Consumer.from("g1", "c1")`</h4>

指定**消费者身份**：

- `"g1"` → 消费者组名称（Group）
- `"c1"` → 消费者名称（Consumer）

对应 SQL 里的 `GROUP g1 c1`。消费者组是消息分发的基本单位——同组内多个消费者分摊消息，不同组各自独立消费全量。

<h4>④ `StreamReadOptions.empty()`</h4>

创建一个**默认的读取选项对象**，然后链式追加两个配置：

<h4>④a `.count(1)`</h4>

最多读 **1 条**消息。对应 `COUNT 1`。

<h4>④b `.block(Duration.ofSeconds(2))`</h4>

如果没有新消息，**阻塞等待 2 秒**。对应 `BLOCK 2000`（单位毫秒）。2 秒内来消息了立刻返回；2 秒还没消息，返回 null。

<h4>⑤ `StreamOffset.create("stream.orders", ReadOffset.lastConsumed())`</h4>

指定**从哪个 Stream 读、从哪个位置读**：

- `"stream.orders"` → Stream 的 key 名
- `ReadOffset.lastConsumed()` → 从「上次消费的位置」往后读，底层对应 `>` 符号

`>` 的含义：**只读「从未投递给任何消费者」的新消息**。已投递但未 ACK 的消息（在 pending-list 里的）不会被 `>` 读到。

<h4>拼起来等价于这条 Redis 命令</h4>

```
XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS stream.orders >
```

<h4>返回值</h4>

`List<MapRecord<String, Object, Object>>` —— 读到的消息列表。每条消息是一个 `MapRecord`，里面包含：

- `getId()` → 消息的 Redis Stream ID（如 `1234567890-0`），后面 ACK 时要用
- `getValue()` → 消息体，一个 Map，里面是 Lua 脚本 XADD 时塞进去的 `userId`、`voucherId`、`id` 三个字段

> if(list==null||list.isEmpty())为什么要判断null和空？直接null或空不行吗？

因为 `read()` 方法返回 `null` 和返回空 List 是**两种不同的情况**，只判一个会漏掉另一个：

| 返回值          | 什么时候出现                          | 只判 null              | 只判 isEmpty           |
| :-------------- | :------------------------------------ | :--------------------- | :--------------------- |
| `null`          | BLOCK 2 秒超时，Redis 没有任何新消息  | ✓ 捕获到               | ✗ NullPointerException |
| `[]`（空 List） | 极端边界：Redis 返回了结构但 0 条消息 | ✗ 漏掉 → `get(0)` 越界 | ✓ 捕获到               |

- **只判 `null`**：如果返回的是空 List（不是 null），`null` 检查通过 → 走到 `list.get(0)` → `IndexOutOfBoundsException`
- **只判 `isEmpty()`**：如果返回的是 `null`，调 `null.isEmpty()` → `NullPointerException`

所以必须 `list == null || list.isEmpty()` 两个都判，短路或 `||` 保证 null 时不会再调 `isEmpty()`。

> 为什么run()方法中用continue,handlePendingList()中用break？

因为两个 `while(true)` 的**职责完全不同**：

**run() —— 永久监听，不能停**

```java
while(true) {
    // 读新消息（BLOCK 2 秒）
    if (list 为空) {
        continue;  // ← 没消息？回去继续等下一轮，永远监听
    }
    // 处理消息
}
```

`continue` 的意思：这一轮没捞到消息，**回到 while 顶部再来一轮**。消费者要 7×24 小时盯着 Stream，没消息就继续阻塞等，不能退出。

**handlePendingList() —— 一次性清理，干完就走**

```java
while(true) {
    // 读 pending-list 中的残留消息
    if (list 为空) {
        break;  // ← pending-list 清空了，退出循环，回调用者
    }
    // 处理残留消息
}
```

`break` 的意思：pending-list 里没有残留消息了，**清理完毕，退出循环**。这个方法是 `run()` 的 catch 块里调的——异常时进来扫一遍残留，扫完就回 `run()` 继续正常监听。如果这里也用 `continue`，pending-list 空了还在死循环转，空转浪费 CPU。

一句话：**run() 是「常驻监听」用 continue 继续等，handlePendingList() 是「临时清理」用 break 干完退出。**

## 达人探店

### 实战篇04(点赞排行榜)

>为什么Boolean类型的`isMember`要使用BooleanUtil来防止空指针，Double类型的`score`就可以直接判断呢？

~~~java
Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
        if(BooleanUtil.isFalse(isMember)){
            ...
        }
~~~

```java
Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
	if(score==null){//?为什么这里不用管拆箱时为null的情况呢？
	...
}
```

==问题的关键在于究竟有没有拆箱操作，判断`==null`或则`!=null`时比较的是**引用本身**是不是空，这种操作不会涉及拆箱，也就不会出现自动拆箱时出现`null.booleanValue()`抛出空指针异常了==

对于第一种情况来说`isMember`本身就是`Boolean`类型的包装类，想要判断它是`true`还是`false`就会涉及拆箱，用`BooleanUtil`可以保证自动拆箱时出现`null`也能判断成`false`

第二种情况本身就只需要判断`score`这个`Double`类型的包装类是否为`null`不涉及拆箱操作，也就不需要工具类来额外辅助了。

WorkBuddy:

**差别不在类型，而在你对这个值做了什么操作**。

<h4>直接判 null 本来就不需要工具类</h4>

先看清楚两种写法各自在干什么：

```java
// Double 的写法：只做了"引用判空"，没有拆箱
Double score = ...;
blog.setIsLike(score != null);        // score 和 null 比地址，永远安全

// Boolean 如果也这样写，同样安全：
Boolean isMember = ...;
if (isMember != null && !isMember) { ... }   // ← 这也完全OK
```

`xxx != null` 判断的是**引用本身**是不是空，不管 `xxx` 是 Boolean、Double 还是 String，都不涉及拆箱，永远不会 NPE。

<h4>危险的是“自动拆箱”</h4>

真正会炸的是把包装类型**当基本类型用**的那一刻：

```java
if (isMember) { ... }
//  ↑ 编译器补了一句 isMember.booleanValue()
//    isMember 为 null 时 → null.booleanValue() → NPE

if (score > 0) { ... }
//  ↑ 同样会补 score.doubleValue()
//    score 为 null 时 → 一样 NPE！
```

所以 Double **并不是天生免疫**——你要是写 `score > 0`，null 时照样炸。它安全只是因为你的业务只需要知道“有没有这个分数”，`!= null` 就够用，没必要拆箱。

<h4>BooleanUtil.isFalse 到底干了什么</h4>

它就是帮你做了 null 安全的判断：

```java
// BooleanUtil.isFalse(isMember) 等价于：
Boolean.FALSE.equals(isMember)
// null → false（不是点赞状态，走点赞分支）
// false → true
// true → false
```

三选一对比：

| 写法                                 | null 时结果 | 说明                             |
| :----------------------------------- | :---------- | :------------------------------- |
| `if (isMember)`                      | **NPE**     | 自动拆箱，null 没有 booleanValue |
| `if (isMember != null && !isMember)` | false       | 安全，但啰嗦                     |
| `BooleanUtil.isFalse(isMember)`      | false       | 安全 + 简洁，语义清晰            |

------

==`if (isMember != null && !isMember)`等价于`BooleanUtil.isFalse(isMember)`==代表==不是 null，并且值是 false==时才能进入if条件。

拆开看，这个条件其实由**两段小判断**组成，靠 `&&` 串起来：

```java
isMember != null   //  第一段：它不是 null 吗？
&&                 //  并且（短路与）
!isMember          //  第二段：它是 false 吗？
```

整体含义：**“不是 null，并且值是 false”** → 也就是“安全地确认它是 false”。

------



<h4>一句话总结</h4>

- **判空**（`!= null`）：任何包装类型都安全，不需要工具类
- **用值**（`if (bool)`、`score > 0`）：触发自动拆箱，null 就 NPE——BooleanUtil 的作用是把 Boolean 的“用值”包装成 null 安全的操作

如果哪天你需要判断“分数是否大于某个值”，`score` 也得先判 null 或者用 `ObjectUtil.defaultIfNull(score, 0.0) > 0` 之类的写法，和 Boolean 是同一个道理。

> 这段代码涉及两个知识点:1、批量查询操作的顺序问题2、Java8新特性Stream流式操作

~~~java
    List<UserDTO> userDTOS = userService.query()
            .in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list()
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
~~~



### 1、MySql==批量查数据==时的的`in`操作导致查出顺序和想要顺序不一致问题

有一条铁律：

> **任何依赖顺序的查询，必须显式的指定排序规则，没有指定排序规则返回顺序视为随机。**

#### Mysql中IN的用法

`IN` 是**罗列具体值**，不是范围。它就是“从这几个值里挑”的意思。

<h5>基本语法</h5>

```sql
SELECT * FROM user WHERE id IN (5, 1);
```

意思是：**id 等于 5，或者等于 1** 的行。它完全等价于一串 OR：

```sql
SELECT * FROM user WHERE id = 5 OR id = 1;
-- 和上面 IN 的写法结果一模一样
```

| 写法              | 含义                       | 类型     |
| :---------------- | :------------------------- | :------- |
| `IN (5, 1)`       | 等于这几个**具体值**之一   | 枚举清单 |
| `BETWEEN 1 AND 5` | 落在 **1 ≤ id ≤ 5** 的区间 | 范围     |
| `id > 4`          | 大于某个值                 | 范围     |

想表达“范围”用的是 `BETWEEN`、`>`、`<` 这些；`IN` 只认清单里逐个列出来的值，1 到 100 万你总不能用 IN 写一百万个值。

#### Java中MyBatis-Plus对应IN的用法

==教程代码中传入了list集合==

<h5>MyBatis-Plus 的 `in` 有两个重载</h5>

```java
in(String column, Collection<?> coll);   // 接集合：List、Set 等
in(String column, Object... values);     // 接可变参数：一个或多个任意值
```

所以这些写法都合法：

```java
.in("id", ids)           // ids 是 List → 走第一个重载 → id IN (?, ?)
.in("id", 5)             // 单个数字 → 走第二个重载 → id IN (?)
.in("id", 5, 1, 3)       // 多个数字 → id IN (?, ?, ?)
```

#### 言归正传

~~~java
userService.query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list()
~~~

这段操作中如果不指定后面的`ORDER BY`的排序规则的话就会导致，虽然查出了前5名的点赞用户，但排序顺序是乱的导致**“按照点赞先后显示”**的功能没有实现

所以要加一段排序规则：`last("ORDER BY FIELD(id," + idStr + ")")`

<h5>一、语法规则</h5>

<h5>1. `last()` 是什么</h5>

`last()` 是 MyBatis-Plus 链式构建器（`QueryChainWrapper`）上的一个方法，作用是：**把你给的字符串原封不动地拼到最终 SQL 的最末尾**。==tip:很显然有SQL注入风险==

```java
.last(String sqlFragment)
```

三个关键特性：

| 特性           | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| **原样拼接**   | 内容不经过任何翻译、不加占位符 `?`，写什么拼什么             |
| **只拼一次**   | 一个链上只应调用一次（多次调用行为不可预期）                 |
| **没有防注入** | 因为是直接拼字符串，如果内容含用户输入，就有 **SQL 注入风险** |

<h5>2. 字符串拼接部分</h5>

```java
"ORDER BY FIELD(id," + idStr + ")"
```

这是普通 Java 字符串拼接。假设 `ids = [5, 1]`，前面循环拼出来的 `idStr = "5,1"`，那么：

```
"ORDER BY FIELD(id," + "5,1" + ")"  →  "ORDER BY FIELD(id,5,1)"
```

<h5>3. 生成的最终 SQL</h5>

```sql
SELECT ... FROM user WHERE id IN (?, ?) ORDER BY FIELD(id,5,1)
```

`last()` 的内容永远排在**整条 SQL 的最后一个位置**（排在 WHERE、GROUP BY 之后）——这也是它叫 "last" 的原因。

<h5>二、具体干了啥</h5>

<h5>`FIELD()` 是 MySQL 的一个内置函数</h5>

```sql
FIELD(值, 清单...)  →  返回“值”在清单中的位置序号，找不到返回 0
```

示例：

```sql
FIELD(5, 5, 1)   →  1    -- 5 是清单里的第 1 个
FIELD(1, 5, 1)   →  2    -- 1 是清单里的第 2 个
FIELD(9, 5, 1)   →  0    -- 不在清单里
```

<h5>`ORDER BY FIELD(id, 5, 1)` 的排序原理</h5>

`ORDER BY` 后面跟一个表达式时，MySQL 按这个表达式的**计算结果**排序。所以逐行计算：

| 行的 id | FIELD(id, 5, 1) 的结果 |
| :------ | :--------------------- |
| 5       | 1                      |
| 1       | 2                      |

意为：每次获取当前行的id,判断这个id在我给定清单里的那个位置，返回这个位置。就能做到返回顺序的按照罗列的清单顺序。

也就是说：==**返回顺序严格服从你罗列的清单顺序**==。

#### 查询多行数据的顺序问题分页查询时就遇到过

~~~java
Page<Blog> page = query()
                .orderByDesc("liked")
                .page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
~~~

这里的`orderByDesc("liked")`代表按照`liked`的值升序排列输出，只是当时没注意。其实早有提示

------

自己整理这个点赞排行榜时由于不会Stream流式操作，所以使用基础的`iterator`迭代器来查询数据，但我的逻辑是==一个一个查询==然后装入list，所以也能做到按点赞先后返回。

~~~java
//1.1解析为存放UserDTO的List
        List<UserDTO> userDTOList = new ArrayList();
        Iterator<String> iterator = top5.iterator();
        while(iterator.hasNext()){
            //获取数据库中的user
            User user = userService.getById(Long.valueOf(iterator.next()));
            //防止用户被注销时，Redis中还保存有点赞记录导致数据库查出来null
            if(user==null){
                continue;
            }
            //user转userDTO
            UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
            //装进List中
            userDTOList.add(userDTO);
        }
~~~

------

当然Stream还是要学的，这样更优雅。下面就是Stream的学习

### 2、Java8特性——Stream流（补充知识）

## 好友关注

### 实战篇09-好友关注-滚动分页查询

在计算完偏移量os后，还要处理一下极端情况：同分极端场景 5 5 | 5 5 | 5，此时max=minTime，导致重复查到第二组的5 5。（课程没说，笔记后补的）

`os = max == minTime ? os + offset : os;`

~~~java
//解决同分极端场景 5 5 | 5 5 | 5，此时max=minTime，导致查到重复的数据
//如果minTime==max，说明最大的时间和计算出来的下次要查的最大时间（即minTime）是相同的，此时如果os还是2的话，就会导致重新查到第二组5 5 ，就这样一直重复查到
//解决办法就是如果出现minTime==max，就把本次计算出来的下次偏移量：“os”和本次传进来的偏移量：“offset”相加即可查到第三组5
os = max == minTime ? os + offset : os;
~~~

<h4>为什么还要 `+ offset`？触发场景：同分爆炸</h4>

但有一种情况 `os` 单独会**死循环**——**所有元素分数都相同**（比如同一毫秒涌进来 100 条 feed）。

数据 `[5, 5, 5, 5, 5]`（5 个5），count=2，单纯用 `os`：

| 调用 | max=5 | offset 入 | 返回   | minTime | os   | 用 `os` 当下次 offset |
| :--- | :---- | :-------- | :----- | :------ | :--- | :-------------------- |
| 1    | 5     | 0         | [5, 5] | 5       | 2    | 2                     |
| 2    | 5     | 2         | [5, 5] | 5       | 2    | 2                     |
| 3    | 5     | 2         | [5, 5] | 5       | 2    | 2                     |
| ...  | ...   | ...       | ...    | ...     | ...  | ...                   |

**永远返回 [5, 5]，永远停不下来**——因为 `minTime == max` 一直在边界上，而 `os` 永远是 2，根本没在累加。已经把 2 个跳过了的事实在下一次调用里被"遗忘"了。

## 附近商铺

### 实战篇13-实现附近商铺

> 为啥要做这个判断

~~~java
 if (list.size() <= from) {
            // 没有下一页了，结束
            return Result.ok(Collections.emptyList());
        }
~~~

<h4>它判断的是：半径内实际有多少家，够不够凑出这一页</h4>

先理解这套分页策略的背景：==**Redis 的 GEOSEARCH 不支持"跳过前 N 条"的偏移分页**==，所以代码的办法是——一次查出**前 end 条**（`limit(end)`），回到内存里再 `skip(from)` 模拟分页。

关键点：`limit(end)` 只承诺"**最多**返回 end 条"。如果半径内总共就 3 家店，它不会给你凑够 10 条，**实际返回条数 = 半径内的真实总数**。

于是判断就变成两个数的比较：

| 场景（每页5条，翻第2页） | from | Redis最多查(end) | 实际返回(list.size) | 判断  | 结果                         |
| :----------------------- | :--- | :--------------- | :------------------ | :---- | :--------------------------- |
| 半径内有 8 家店          | 5    | 10               | 8                   | 8 > 5 | 跳过前5，返回第6~8家（3条）✓ |
| 半径内只有 3 家店        | 5    | 10               | 3                   | 3 ≤ 5 | **第2页不存在，返回空** ✓    |

第二行就是判断存在的意义：**from 是"本页从第几条开始取"的下标，如果半径内的真实总数连这个起点都够不到，说明已经翻过最后一页了**，本页必然没数据。

<h4>如果不做这个判断会怎样？——直接崩<?h4>

这才是关键。试想半径内只有 3 家店，用户翻到第 2 页，没有这个判断，代码会继续往下走：

```java
list.stream().skip(from)   // 3 条里 skip 5 条 → 一条不剩
ids = []                   // 空列表
idStr = StrUtil.join(",", ids)  // → ""，空字符串！
query().in("id", ids)
       .last("ORDER BY FIELD(id," + "" + ")")  // → ORDER BY FIELD(id,)  ← 非法 SQL！
```

拼出来的 SQL 是 `WHERE id IN () ORDER BY FIELD(id,)`——MySQL 里**两条都是语法错误**，直接抛异常。所以这个判断不是"可选的优化"，而是**防止空列表继续往下走、生成非法 SQL 的兜底**，顺带也省了一次无意义的数据库查询。

<h4>和传统分页的对照</h4>

它在语义上等同于 SQL 分页里的越界判断：

```java
// 传统数据库分页：总条数 < (current-1)*size  → 已经翻过头了
if (total <= from) {
    // 没有更多数据了
}
```

区别只是这里的"总数"是 Redis 半径查询的实际返回数，而且 `limit(end)` 本身就限制了最多只能拿到 end 条，所以判断的是 `list.size()`（真实条数）和 `from`（本页起点）的关系。

<h4>为什么是 `<=` 而不是 `<`？</h4>

因为下标从 0 开始。`list.size() == from` 时，意味着正好从第 from 条开始就**没有**了——比如总共 5 家、翻第 2 页（from=5），第 2 页一条都取不到，必须判空。用 `<=` 把这个"恰好到边界"的情况也兜住。

一句话：**这个判断在回答"半径内真实返回的条数，是否够得着本页的起点下标"——够不着就说明这一页是空的，提前返回空列表；不做它，空列表继续往下就会拼出 `ORDER BY FIELD(id,)` 这种非法 SQL 直接报错。**

> BUG:用Stream toMap()时排序被吞

~~~java
 Map<Long, @NonNull Distance> distanceMap = list.stream()
                .skip(from)
                .collect(Collectors.toMap(result -> Long.valueOf(result.getContent().getName())
                        , result -> result.getDistance(),(a,b)->a));

        //获取id集合
        /* List<Long> ids=new ArrayList<>(list.size());
        for (Long id : distanceMap.keySet()) {
            ids.add(id);
        }*/
        //用重构方法更简单
        ArrayList<Long> ids = new ArrayList<>(distanceMap.keySet());
        //4.2查询数据库
        String idStr = StrUtil.join(",", ids);
        List<Shop> shops = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
        //4.3 对应距离装进数据库
        for (Shop shop : shops) {
            Long shopId = shop.getId();
            shop.setDistance(distanceMap.get(shopId).getValue());
        }
        //5.返回shop
        return Result.ok(shops);
~~~

![image-20260904221519588](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20260904221519588.png)

解决办法（有很多，举两个例子）：

<h4>方案 1：让 toMap 用 LinkedHashMap（保留插入顺序）</h4>

`Collectors.toMap` 有第四个参数，指定用哪种 Map 实现：

java

```java
Map<Long, Distance> distanceMap = list.stream()
    .skip(from)
    .collect(Collectors.toMap(
        result -> Long.valueOf(result.getContent().getName()),
        result -> result.getDistance(),
        (a, b) -> a,				// key重复时保留先出现的value
        LinkedHashMap::new          // ← 第4个参数：按插入顺序的 Map
    ));
```

`LinkedHashMap` 在 HashMap 基础上多维护一条**双向链表**记录插入顺序，`keySet()` 迭代时按插入顺序吐出——2, 9, 8, 3, 6 原样保留。

<h4>方案2教程提供的通过ForEach顺序遍历的同时将id按顺序存进去</h4>

~~~java
List<Long> ids = new ArrayList<>(list.size());
Map<String, Distance> distanceMap = new HashMap<>(list.size());
list.stream().skip(from).forEach(result -> {
            // 4.2.获取店铺id
            String shopIdStr = result.getContent().getName();
            ids.add(Long.valueOf(shopIdStr));
            // 4.3.获取距离
            Distance distance = result.getDistance();
            distanceMap.put(shopIdStr, distance);
        });
        // 5.根据id查询Shop
        String idStr = StrUtil.join(",", ids);
~~~

==教程直接遍历不用tomap都是有原因的，当然用`LinkedHashMap::new `也能解决==

## 过滤器与拦截器（补充知识）



