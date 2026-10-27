# Redis知识点速查

## 1、短信登录

### 1.1短信验证登录

#### 需求：用户登录发送验证码，登录

![image-20261018214719680](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018214719680.png)

#### 问题：

验证码这种失效性很短的消耗品存入MySQL很浪费内存,而且不能设置过期时间得手动清除验证过的验证码

#### 解决：

1. 使用redis存储验证码，可以同时设置过期时间，自动清除验证过的验证码，节省空间
2. redis基于内存验证时比MySQL更快

**存储验证码到redis**

~~~java
public Result sendCode(String phone, HttpSession session) {
        //1.校验手机号格式
        if (RegexUtils.isPhoneInvalid(phone)) {//RegexUtils.isPhoneInvalid(phone)为true代表手机号无效
            //2.不符合返回错误信息
            return Result.fail("手机号格式错误");
        }
        //3.符合生成验证码
        String code = RandomUtil.randomNumbers(6);
        //4.保存验证码到session
        //session.setAttribute("code",code);
        //4.保存验证码到Redis,设置数据有效期
        stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY + phone, code, LOGIN_CODE_TTL, TimeUnit.MINUTES);
        //5.发送验证码
        log.debug("发送短信验证码成功,验证码：{}", code);
        //6.返回ok
        return Result.ok();
    }
~~~

**获取验证码验证**

~~~java
 //3.从redis中获取验证码并校验
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
        String code = loginForm.getCode();
        if (cacheCode == null || !cacheCode.toString().equals(code)) {
            //3.1不一致报错
            return Result.fail("验证码错误");
        }
~~~

### 1.2验证登录状态

#### 需求：验证用户登录状态，并延长登录状态

![image-20261018213241843](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018213241843.png)

#### 问题:

Session是单体架构无法满足跨机器验证用户登录状态的需求

#### 解决:

1. 登录注册时：登录注册成功后（验证码通过），redis中key存储token，value存储用户信息。其中token返回前端，前端将token存入`authorization`**请求头**中。
2. 验证时（拦截器处验证）：通过发送时获取**请求头**中的token来redis中验证用户登录状态。token为空，未登录或者登录过期。token非空，用户在线，查询redis中用户信息并存入`ThreadLocal`中以便业务链路取用、刷新过期时间以演唱登录状态。

**生成token,存用户信息进redis**

~~~java
 String token = UUID.randomUUID().toString();
        //5.2User->UserDTO,UserDTO转化成HashMap存储
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(), CopyOptions.create().
                setIgnoreNullValue(true).setFieldValueEditor((fieldName, fieldValue) -> fieldValue.toString()));
        //5.3将tokenkey存入redis
        String tokenKey = LOGIN_USER_KEY + token;
        stringRedisTemplate.opsForHash().putAll(tokenKey, userMap);
        //5.4设置有效期
        stringRedisTemplate.expire(tokenKey, LOGIN_USER_TTL, TimeUnit.SECONDS);
        //5.5返回token
        return Result.ok(token);
~~~

**拦截器处验证登录状态**

~~~java
//1.获取请求头中的token
        String token = request.getHeader("authorization");
        //1.1判断token，为空拦截
        if(StrUtil.isBlank(token)){
            return false;
        }
        //2.通过token获取用户
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(LOGIN_USER_KEY + token);
        //2.1判断用户是否存在，不存在拦截。双重验证，
        if(userMap.isEmpty()){
            return false;
        }
        //3.将查到的Hash数据转成UserDTO
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        //4.存在保存用户到ThreadLocal
        UserHolder.saveUser(userDTO);
        //5.刷新token有效期
        stringRedisTemplate.expire(LOGIN_USER_KEY + token,LOGIN_USER_TTL, TimeUnit.SECONDS);
		return true;
~~~

**配置拦截器生效**

~~~java
package com.hmdp.config;

import com.hmdp.utils.LoginInterceptor;
import com.hmdp.utils.RefreshTokenInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).excludePathPatterns(
                "/user/code",
                "/user/login",
                "/shop/**",
                "/shop-type/**",
                "/voucher/**",
                "/upload/**",
                "/blog/hot"
        ).order(1);
    }
}

~~~

### 1.3双拦截器

#### 先问题：

拦截器只能拦截指定页面并验证登录状态，哪些不需要验证登录状态的页面虽然不需要登录，**但如果一个已经登录的用户点进了无需验证登录状态的页面过久导致key过期从而登录状态失效。此时用户需要进入需要验证登录状态的页面发现自己无法访问了，很影响用户体验。**

#### 后需求：所有页面都应该有延长登录状态的功能，且无需登录的界面依然无需验证登录状态。

#### 解决：

此时一个拦截器无法满足需求，用两个拦截器来分化需求。

- 拦截器1拦截所有请求，但这个拦截并不实际拦截请求。他的作用只有一个：刷新用户登录状态，并存储用户信息到ThreadLocal中。无论有没有用户登录，他都不拦截。
- 拦截器2拦截需要验证登录态的请求，这里同样也可以获取token并验证，但是效率更高的方式是：由于拦截器1已近将登录过的用户信息存入到ThreadLocal中了。**直接验证是否能获取ThreadLocal中的用户**，获取到则登录了，获取不到则未登录。省去了查redis的操作，更高效。

**拦截器1**

~~~java

public class RefreshTokenInterceptor implements HandlerInterceptor {

    private StringRedisTemplate stringRedisTemplate;

    public RefreshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1.获取请求头中的token
        String token = request.getHeader("authorization");
        //1.1判断token，即使为空也不拦截
        if(StrUtil.isBlank(token)){
            return true;
        }
        //2.通过token获取用户
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(LOGIN_USER_KEY + token);
        //2.1判断用户是否存在，即使不存在也不拦截
        if(userMap.isEmpty()){
            return true;
        }
        //3.将查到的Hash数据转成UserDTO
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        //4.存在保存用户到ThreadLocal
        UserHolder.saveUser(userDTO);
        //5.刷新token有效期
        stringRedisTemplate.expire(LOGIN_USER_KEY + token,LOGIN_USER_TTL, TimeUnit.SECONDS);
        //6.放行
        return true;
    }
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        UserHolder.removeUser();
    }
}

~~~

**拦截器2**

~~~java
public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //1.判断是否需要拦截（依据是ThreadLocal是否为空）
        if (UserHolder.getUser()==null) {
            //为空则拦截
            response.setStatus(401);
            return false;
        }
        //放行
        return true;
    }

}

~~~

**配置拦截器生效**

~~~java
package com.hmdp.config;

import com.hmdp.utils.LoginInterceptor;
import com.hmdp.utils.RefreshTokenInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).excludePathPatterns(
                "/user/code",
                "/user/login",
                "/shop/**",
                "/shop-type/**",
                "/voucher/**",
                "/upload/**",
                "/blog/hot"
        ).order(1);
        registry.addInterceptor(new RefreshTokenInterceptor(stringRedisTemplate)).addPathPatterns("/**").order(0);
    }
}
~~~

## 2、商户缓存

### 2.1缓存商户信息

#### 问题：对于商铺这种热点数据，查数据库太慢。

#### 需求：快速获取热点数据

#### 解决：

使用redis缓存热点数据，查商铺时先过redis,redis中查不到再查数据库，**查到了再写入数据库中（数据同步）**。

![image-20261018230022222](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018230022222.png)

这里应该先**缓存预热**一下，但教程为方便其实直接走了查不到->写入缓存的步骤。按理说**查不到->写入缓存**主要目的是为了做**数据同步**的。

![image-20261018230450750](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018230450750.png)

### 2.2 双写一致

#### 需求：数据库中商户信息更新，需同步到缓存中

#### 问题：

由于我们的**缓存的数据源来自于数据库**,而数据库的**数据是会发生变化的**,因此,如果当数据库中**数据发生变化,而缓存却没有同步**,此时就会有**一致性问题存在**,其后果是:用户使用缓存中的过时数据,就会产生类似多线程数据安全问题,从而影响业务,产品口碑等;

#### 解决：

有如下几种方案

1. Cache Aside Pattern 人工编码方式：缓存调用者在更新完数据库后再去更新缓存，也称之为**双写方案**

2. Read/Write Through Pattern : 由系统本身完成，数据库与缓存的问题交由系统本身去处理

3. Write Behind Caching Pattern ：调用者只操作缓存，其他线程去异步处理数据库，实现最终一致（秒杀业务）

**1、用第一种双写一致方案**

更新数据库再更新缓存，但假设我们每次操作数据库后，都操作缓存，但是中间如果没有人查询，那么这个更新动作实际上只有最后一次生效，中间的更新动作意义并不大，我们可以把**缓存删除**，等待再次查询时，将缓存中的数据加载出来（即查询时（2.1）做的操作发现**缓存不存在时再写缓存**）

得出结论：**更新数据库，删缓存**

**2、先操作缓存还是先操作数据库？**

* 先删除缓存，再操作数据库
* **先操作数据库，再删除缓存（√）**

![image-20261018231938770](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018231938770.png)

得出结论：**先更新数据库，再删缓存**



核心思路如下：

修改ShopController中的业务逻辑，满足下面的需求：

根据id查询店铺时，如果缓存未命中，则查询数据库，将数据库结果写入缓存，并设置超时时间

根据id修改店铺时，先修改数据库，再删除缓存

**修改重点代码1**：修改**ShopServiceImpl**的queryById方法

**设置redis缓存时添加过期时间**

![image-20261018232418710](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018232418710.png)

**修改重点代码2**

代码分析：通过之前的淘汰，我们确定了采用删除策略，来解决双写问题，当我们修改了数据之后，然后把缓存中的数据进行删除，查询时发现缓存中没有数据，则会从mysql中加载最新的数据，从而避免数据库和缓存不一致的问题

![image-20261018232444927](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018232444927.png)

### 2.3延迟双删

#### 问题：**"先更DB后删缓存"也有一个极端的并发漏洞**

它并非 100% 无懈可击，经典反例是**"读请求查库慢、更新请求删缓存快"**：

```text
时刻   线程A(查询,缓存未命中)        线程B(更新)
 t1    查缓存→未命中
 t2                                   更新DB → 20
 t3                                   删除缓存
 t4    查DB→读到【旧值10】(A在B更新前就发起了查库)
 t5    把【旧值10】写回缓存           缓存=10 ❌
```

**条件很苛刻**：读线程A在线程B更新之前读到旧值，且直到B线程更新数据库加删缓存操作完后，线程A才执行到写缓存的操作，此时缓存是旧值，数据库新值。

换成耗时比较，就是：

> **线程A从"查缓存未命中"到"写回缓存"的这段耗时 > 线程B从"更新DB"到"删缓存"的耗时**

tip：想想就很苛刻，首先你得再更新数据库前查到旧值，更逆天的是一个查缓存的操作要比更新数据库加删缓存更慢？太极端了。

#### 解决：

**延迟双删除：**删除缓存后，过一小段时间再删除一次。

**先更新DB → 删缓存 → 延迟 → 再删一次缓存**

```text
t1  更新数据库 → 20
t2  删除缓存
t3  ...延迟...
t4  再删一次缓存   ← 兜底清掉那种慢查询回填的旧值
```

### 2.4缓存穿透

#### 问题：

#### 缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库。

#### 解决：

常见的解决方案有两种：

* 缓存空对象
  * 优点：实现简单，维护方便
  * 缺点：
    * 额外的内存消耗
    * 可能造成短期的不一致
* 布隆过滤
  * 优点：内存占用较少，没有多余key
  * 缺点：
    * 实现复杂
    * 存在误判可能

![image-20261018235609000](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261018235609000.png)

**缓存空对象：**查到不存在数据时，给这个不存在数据存一个value为空的数据放入redis中，这样下次访问时缓存直接返回空数据保护数据库。

~~~java
public Result queryById(Long id) {
    //1.Redis查询商铺缓存
    String shopJson = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY + id);
    //2.判断是否存在
    if (StrUtil.isNotBlank(shopJson)) {
        //3.存在，json转对象直接返回
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        return Result.ok(shop);
    }
    //4.判断是否为空值
    if(shopJson!=null){
        return Result.fail("店铺不存在");
    }
    //5.若为null则不存在
    //5.1查询数据库
    Shop shop = getById(id);
    if (shop == null) {
        //5.1.1 商铺不存在,将空值写入Redis中并设定TTL，返回错误信息
    stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+id,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
        return Result.fail("店铺不存在");
    }
    //5.1.2 存在商铺数据写入Redis
    stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id,JSONUtil.toJsonStr(shop),CACHE_SHOP_TTL, TimeUnit.MINUTES);
    //6.返回商铺信息
    return Result.ok(shop);
}
~~~



**布隆过滤器：**就是在请求缓存之前再加一层布隆过滤器，这个过滤器提前预热了redis中的key,如果返回false证明这个数据一定不存在，直接返回错误，甚至redis都不走。如果返回true，证明这个数据可能存在再往下走。（往下走缓存空值依然可以用于判断。）

关于布隆过滤器的原理其实不难，就是通过三个哈希函数将数据是否存在的状态映射到位属组对应位置的1或0状态。
具体看https://javaguide.cn/cs-basics/data-structure/bloom-filter.html		

布隆过滤器有现成的组件可以用来很快捷的调用，要做的只是调用对应的函数预热数据即可。比如

1、单体项目中（jvm范围），Guava提供的布隆

2、分布式项目中，Redison提供的布隆

### 2.5缓存雪崩

#### 问题：

同一时段大量的缓存key同时失效或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力。

#### 解决：

* 给不同的Key的TTL添加随机值
* 利用Redis集群提高服务的可用性==高级篇==
* 给缓存业务添加降级限流策略==SpringCloud==
* 给业务添加多级缓存==用SpringCloud解决多级缓存一致性问题（ **JetCache（阿里，推荐）**）==

### 2.6缓存击穿

#### 问题：

缓存击穿问题也叫热点Key问题，就是一个==被高并发访问并且缓存重建业务较复杂的key==突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击。

#### 解决：

常见的解决方案有两种：

* 互斥锁
* 逻辑过期

**互斥锁：**缓存失效时，串行化查询查询数据库并重建缓存，其他线程未获取到互斥锁自旋重新查缓存，获取互斥锁

![image-20261019001804216](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261019001804216.png)

~~~java
public Shop queryWithMutex(Long id){
        //1.Redis查询商铺缓存
        String shopJson = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY + id);
        //2.判断是否存在
        if (StrUtil.isNotBlank(shopJson)) {
            //3.存在，json转对象直接返回
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            return shop;
        }
        //4.判断是否为空值
        if(shopJson!=null){
            return null;
        }
        //5.若为null则不存在,实现缓存重构
        //5.1 先获取互斥锁
        Shop shop = null;
        String lockKey=LOCK_SHOP_KEY+id;
        //5.2 获取锁并判断是否获取成功
        try {
            boolean isLock = tryLock(lockKey);
            if(!isLock){
                //5.3 获取失败,休眠一会继续递归获取
                Thread.sleep(50);
                return queryWithMutex(id);
            }
            //5.5 获取成功
            //加个延迟测试一下互斥锁实现缓存重建
            //Thread.sleep(200);
            //5.6 获取成功后再次检查Redis缓存是否存在，若存在则无需重建缓存
            shopJson=stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY + id);
            if(StrUtil.isNotBlank(shopJson)){
                //存在，json转对象直接返回,这里try了整个if,不用额外手动释放
                return JSONUtil.toBean(shopJson,Shop.class);
            }
            if(shopJson!=null){
                //为空值返回null，这里try了整个if,不用额外手动释放
                return null;
            }
            //5.7 为null,重建缓存
            shop = getById(id);
            if (shop == null) {
                //5.7.1 商铺不存在,将空值写入Redis中并设定TTL，返回错误信息
                stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+id,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
                return null;
            }
            //5.7.2 存在商铺数据写入Redis
            stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id,JSONUtil.toJsonStr(shop),CACHE_SHOP_TTL, TimeUnit.MINUTES);

        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }finally {
            //5.8 释放锁
            unLock(lockKey);
        }
        return shop;
    }
~~~



**逻辑过期**：允许获取锁和没获取锁的线程先返回一段时间旧值，获取锁的线程开启异步线程重建缓存，再释放锁。

![image-20261019002050595](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019002050595.png)

~~~java
public Shop queryWithLogicalExpire(Long id){
        String key=CACHE_SHOP_KEY + id;
        //1.Redis查询商铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        //2.判断缓存是否存在
        if (StrUtil.isBlank(json)) {
            //3.不存在直接返回null
            return null;
        }
        //4.存在，先把json反序列化成对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        //4.1 提取具体的shop对象？
        //使用Shop shop = (Shop)redisData.getData();类型不匹配
        Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
        //4.2 提取具体的过期时间
        LocalDateTime expireTime = redisData.getExpireTime();
        //5.判断缓存是否过期
        if (expireTime.isAfter(LocalDateTime.now())) {
            //5.1 未过期直接返回
            return shop;
        }
        //6.已过期重建缓存
        //6.1获取锁
        String lockKey=LOCK_SHOP_KEY+id;
        boolean isLock = tryLock(lockKey);
        //6.2判断锁是否成功
        if (isLock) {
            //获取成功依然要做doubleCheck，检查此时缓存有没有被别的线程提前建立好了
                //Redis查询商铺缓存
                json = stringRedisTemplate.opsForValue().get(key);
                //判断缓存是否存在
                if (StrUtil.isBlank(json)) {
                    //手动释放锁
                    unLock(lockKey);
                    return null;
                }
                //存在，先把json反序列化成对象
                redisData = JSONUtil.toBean(json, RedisData.class);
                //提取具体的shop对象
                shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
                //获取具体的过期时间
                expireTime=redisData.getExpireTime();
                //检查此时的缓存的有没有被更新
                if(expireTime.isAfter(LocalDateTime.now())){
                    //更新,返回shop，手动释放锁
                    unLock(lockKey);
                    return shop;
                }

            //6.3获取成功，开启独立线程重建缓存？
            //这里使用线程池来执行，性能更高
            CACHE_REBUILD_EXECUTOR.submit(()->{
                try {
                    this.saveShop2Redis(id,20L);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }finally {
                    //这个锁的释放时机是保证缓存重建完成后释放锁
                    unLock(lockKey);
                }
            });

        }
        //7.
        return shop;

    }
~~~

## 3、优惠卷秒杀

### 3.1全局唯一ID

#### 需求：

当用户抢购时，就会生成订单并保存到tb_voucher_order这张表中，而订单表如果使用数据库自增ID就存在一些问题：

* id的规律性太明显
* 受单表数据量的限制

场景分析一：如果我们的id具有太明显的规则，用户或者说商业对手很容易猜测出来我们的一些敏感信息，比如商城在一天时间内，卖出了多少单，这明显不合适。

场景分析二：随着我们商城规模越来越大，mysql的单表的容量不宜超过500W，数据量过大之后，我们要进行拆库拆表，但拆分表了之后，他们从逻辑上讲他们是同一张表，所以他们的id是不能一样的， 于是乎我们需要保证id的**全局唯一性**。

#### 解决

利用redis生成ID可以保证全局性，利用 **"时间戳 + 序列号"**保证唯一性。

**tip:**秒级时间戳只保证"每秒不同"，不保证"每次不同"；拼上同秒自增的 `count` 才能做到"每次调用都唯一"。

~~~java
package com.hmdp.utils;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;

@Component
public class RedisIdWorker {
    /**
     * 开始时间戳
     */
    private static final long BEGIN_TIMESTAMP=1785715200;
    /**
     * 序列号位数
     */
    private static final int COUNT_BITS=32;

    private StringRedisTemplate stringRedisTemplate;

    public RedisIdWorker(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public long nextId(String keyPrefix){
        //1.生成时间戳
        LocalDateTime now = LocalDateTime.now();
        //以秒的格式显示
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp=nowSecond-BEGIN_TIMESTAMP;
        //2.生成序列号
        // 2.1.获取当前日期，精确到天
        //当下时间转字符串格式并定义格式化规则
        String data = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        //2.2.自增长
        Long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + data);
        //3.拼接返回
        return timestamp << COUNT_BITS | count;
    }
    //获取指定开始时间
    public static void main(String[] args) {
        LocalDateTime time = LocalDateTime.of(2026, 8, 3, 0, 0, 0);
        long beginsecond = time.toEpochSecond(ZoneOffset.UTC);
        System.out.println("second = " + beginsecond);

    }
}
~~~

### 3.2优惠卷秒杀下单

#### 需求：

秒杀下单应该思考的内容：

下单时需要判断两点：

* 秒杀是否开始或结束，如果尚未开始或已经结束则无法下单
* 库存是否充足，不足则无法下单

#### 解决:(库存超卖问题未解决)

~~~java
@Override
public Result seckillVoucher(Long voucherId) {
    // 1.查询优惠券
    SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
    // 2.判断秒杀是否开始
    if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀尚未开始！");
    }
    // 3.判断秒杀是否已经结束
    if (voucher.getEndTime().isBefore(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀已经结束！");
    }
    // 4.判断库存是否充足
    if (voucher.getStock() < 1) {
        // 库存不足
        return Result.fail("库存不足！");
    }
    //5，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update();
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //6.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 6.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 6.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 6.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);

}
~~~

### 3.3库存超卖

#### 问题：

判断库存时没有任何并发处理，导致==虽然更新数据库那边有行锁保证串行，但判断库存这个动作依然有并发问题。==

假设线程1过来查询库存，判断出来库存大于1，正准备去扣减库存，但是还没有来得及去扣减，此时线程2过来，线程2也去查询库存，发现这个数量一定也大于1，那么这两个线程都会去扣减库存，最终多个线程相当于一起去扣减库存，此时就会出现库存的超卖问题。

![image-20261019104512525](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019104512525.png)

#### 解决

两个方案：

- 乐观锁
- 悲观锁

![image-20261019104711689](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019104711689.png)

**乐观锁解决库存超卖问题**

**版本一：**标准乐观锁,用版本号精准记录判断当下操作时库存有没有被别的线程改过

~~~java
boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0);
~~~

方案一的完整流程（每个线程）：

关键点：`voucher.getStock()`是非数据库操作的业务，不是和数据库操作同时进行的。

```text
T1: 查询库存 voucher.getStock()  → 读到 100        ← 无锁，可以并发读
T2: 业务处理（可能耗时）
T3: UPDATE ... WHERE stock = 100  ← 这里才加行锁、串行执行
```

100 个线程几乎同时执行 T1，**全部读到 100**。然后排队执行 T3：

```text
线程1: UPDATE ... WHERE stock=100 → 命中，stock 变成 99，成功
线程2: UPDATE ... WHERE stock=100 → 此时 stock=99，条件不满足 → 影响行数 0，失败
线程3: 同上 → 失败
... 后面 98 个全失败
```

**UPDATE 确实是串行的**，但每个线程 WHERE 里比较的是**自己在 T1 读到的 100**。等它真正拿到锁执行 UPDATE 时，库存早被别人改成 99 了，条件自然不成立。

**版本二：**

之前的方式要修改前后都保持一致，但是这样成功的概率太低（多个线程同时），所以我们的乐观锁需要变一下，改成stock大于0 即可

扣减库存时直接进行库存余量判断，这样==判断库存和扣减库存就是原子操作==了，结合行锁实现库存扣减。

~~~java
boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0);
~~~

**悲观锁解决库存超卖**：

（此时不推荐）。给整个方法加锁，判断库存+扣减库存+其他业务整段串行

~~~java
public  Synchronized Result seckillVoucher(Long voucherId) {
    // 1.查询优惠券
    SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
    // 2.判断秒杀是否开始
    if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀尚未开始！");
    }
    // 3.判断秒杀是否已经结束
    if (voucher.getEndTime().isBefore(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀已经结束！");
    }
    // 4.判断库存是否充足
    if (voucher.getStock() < 1) {
        // 库存不足
        return Result.fail("库存不足！");
    }
    //5，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1").update();
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //6.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 6.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 6.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 6.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);

}
~~~

### 3.4一人一单

#### 需求：

修改秒杀业务，要求同一个优惠券，一个用户只能下一单

具体操作逻辑如下：比如时间是否充足，如果时间充足，则进一步判断库存是否足够，然后再根据优惠卷id和用户id查询是否已经下过这个订单，如果下过这个订单，则不再下单，否则进行下单。

#### 问题：

直接这样写肯定还是有并发问题，特别是`select`操作是默认没有mysql锁的，也就是说这个操作可以并发的读，然后通过校验减库存。

**tip**:即使你给`select`加了排他锁（select...for update），在减库存没执行前，依然可以有多个线程串行化通过校验然后减库存，导致一人买多单。

~~~java
// 5.一人一单逻辑
    // 5.1.用户id
    Long userId = UserHolder.getUser().getId();
    int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
    // 5.2.判断是否存在
    if (count > 0) {
        // 用户已经购买过了
        return Result.fail("用户已经购买过一次！");
    }
~~~

#### 解决：

##### 解决一：给全方法加个`synchronized`

##### 问题一：开销太大，速度太慢不推荐

~~~java
public Synchronized Result seckillVoucher(Long voucherId) 
~~~

##### 解决二：为解决问题一，用更细粒度的锁

锁住userId的对应的常量池中的唯一数据,并在外面加个事务

~~~java
@Transactional
public  Result createVoucherOrder(Long voucherId) {
	Long userId = UserHolder.getUser().getId();
	synchronized(userId.toString().intern()){
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

##### 问题二：@Transtional注解放在了==jvm锁==的外面，可能出现锁已释放但事务未提交

其他线程拿到锁时看到还是旧数据，又扣减一次，并发又出现了

tip:有关这个问题详看我 （库存超卖引发的事务、Mysql锁以及jvm锁的辨析思考.md）

##### 解决三：为解决问题二，将锁放在事务外面

![image-20261019113028609](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019113028609.png)

##### 问题三：此时用this调用该方法，未经过代理导致事务失效

##### 解决四：用代理来调用方法，事务生效。

![image-20261019113139853](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019113139853.png)

##### 解决五：优化，可以在订单表中将user_id和voucher_id建立一个多列唯一索引做最终==兜底==（兜底乐观锁）

一定要注意==兜底==这个字眼，像这种冲突频率极高的场景只用唯一索引效率是很低的，只能说为了以防万一筑构的最后防线

##### 问题五：jvm锁是单体的，集群环境下失效

用分布式锁解决



#### 这里可以用乐观锁来校验一人一单吗？用乐观锁的条件是什么？

##### 一、先回答：可以，但要看"一人一单"的并发冲突形态

"一人一单"的本质是：**同一个用户对同一个优惠券，只能成功一次**。这是一个"**唯一性约束**"问题。

用乐观锁（版本号/CAS）校验它，有几种可行写法，但各有前提：

###### 写法 A：用户维度加版本号/CAS（不推荐）

给用户-券关系表加一个 `version` 或状态字段，提交时 `WHERE user_id=? AND voucher_id=? AND version=?`。

- **问题**：第一次下单时"记录还不存在"，没有版本号可比，CAS 无从下手。你只能先 insert 再 update，或者用"状态位"做 CAS（比如 `status=0` 才能改成 `1`）。
- **本质**：这其实变成了**"用状态字段做 CAS 的条件更新"**，不是传统版本号乐观锁。

###### 写法 B：唯一索引 + 插入（这才是"一人一单"的正解）

```sql
-- 订单表加唯一索引
UNIQUE KEY uk_user_voucher (user_id, voucher_id)
```



```sql
INSERT INTO voucher_order (user_id, voucher_id, ...) VALUES (...)
-- 重复插入直接抛 DuplicateKeyException
```



- 这是**数据库层面的原子约束**，本质是"**唯一性乐观锁**"——不提前加锁，靠唯一索引在写入时冲突检测。
- 并发下只有一个 insert 成功，其余全部失败，天然幂等、天然原子。
- **这是秒杀"一人一单"最主流、最可靠的做法。**

###### 写法 C：状态位 CAS（可以，但要配合唯一索引或分布式锁）

```sql
UPDATE ... SET status = 1 
WHERE user_id = ? AND voucher_id = ? AND status = 0
```



影响行数为 1 才算抢到。可行，但**前提是记录已存在**，且并发极高时大量失败（同方案一的 CAS 快照问题会重演，不过这里是"状态"不是"库存"，通常可接受）。

###### 写法 D：分布式锁（Redisson）+ 查库判断

先 `tryLock("order:" + userId)`，拿到锁后查"是否已下单"，没有再扣库存建单。这是**悲观锁思路**，不是乐观锁，但秒杀里很常用（配合方案二扣库存）。

##### 二、结论：一人一单的首选不是"版本号乐观锁"

| 方案              | 是否乐观锁         | 推荐度 | 说明                       |
| :---------------- | :----------------- | :----- | :------------------------- |
| 唯一索引 + INSERT | 是（唯一性乐观锁） | ⭐⭐⭐⭐⭐  | 首选，原子、幂等、简单     |
| 状态位 CAS        | 是（条件更新）     | ⭐⭐⭐    | 需记录已存在，高并发失败多 |
| 版本号 CAS        | 是                 | ⭐      | 首次无版本号，不适用       |
| 分布式锁 + 查库   | 否（悲观）         | ⭐⭐⭐⭐   | 秒杀常用，与扣库存配合     |

**推荐组合**：`分布式锁（用户维度）+ 唯一索引兜底`，或直接用 `唯一索引 + INSERT`，扣库存用方案二（`stock > 0`）。乐观锁更适合"**库存**"这种"已有记录、字段可比较"的场景，不太适合"**首次创建、无记录可比**"的一人一单。

##### 三、什么时候能用乐观锁？判断标准

乐观锁适用需要同时满足几个条件：

###### 1. 冲突概率低（读多写少）

乐观锁不提前加锁，靠提交时检测冲突。**冲突少时**它比悲观锁轻量（无锁开销、吞吐高）。冲突极高时（如 100 线程抢 1 库存），大量 CAS 失败要重试，反而更差。

###### 2. 有"可比较的基准值"

需要版本号、时间戳或某个字段的旧值做 compare。**记录必须已存在**，且能拿到"读时的快照"。库存扣减（`stock` 字段）满足；首次创建订单（无记录）不满足。

###### 3. ==冲突检测和执行能在一个原子操作内完成（这句很精髓，用时判断）==

典型就是**单条 SQL 的 `WHERE 旧值 AND SET 新值`**（数据库行锁保证原子），或 **CAS 指令**（CPU 原子指令）。如果"判断"和"修改"分离成两步且中间可被打断，就不是真乐观锁。

###### 4. 失败可接受 / 可重试

乐观锁失败通常返回"请重试"或直接拒绝。业务上要能接受这种失败语义（秒杀里失败=没抢到，可接受；但转账这种就不能随便失败，得重试或换悲观锁）。

##### 四、乐观锁 vs 悲观锁 选择速查

| 场景特征     | 选乐观锁                                  | 选悲观锁                               |
| :----------- | :---------------------------------------- | :------------------------------------- |
| 冲突概率     | 低                                        | 高                                     |
| 读/写比例    | 读多写少                                  | 写多                                   |
| 记录是否存在 | 已存在、可比较                            | 无所谓                                 |
| 失败容忍度   | 可失败可重试                              | 不允许失败                             |
| 典型例子     | 库存扣减（`stock>0`）、状态流转、点赞计数 | 一人一单（锁用户）、账户转账、库存预占 |

##### 五、一句话总结

- **一人一单可以用乐观锁**，但最佳形态是**唯一索引（唯一性乐观锁）+ INSERT**，而不是版本号 CAS（首次无版本号可比）。
- **乐观锁适用**：冲突低、有可比较基准、判断+修改原子、失败可接受。库存扣减（方案二）正是典型；一人一单更适合"唯一约束 + 分布式锁"组合。
- 秒杀生产级方案通常是：**Redis 预扣 + 分布式锁（用户维度）+ 唯一索引兜底 + 数据库 `stock>0` 原子扣减**，多层配合，而非单靠某一种锁。

## 4、分布式锁

### 自研分布式锁

实现分布式锁时需要实现的两个基本方法：

* 获取锁：

  * 互斥：确保只能有一个线程获取锁
  * 非阻塞：尝试一次，成功返回true，失败返回false

* 释放锁：

  * 手动释放
  * 超时释放：获取锁时添加一个超时时间

我们利用redis 的setNx 方法，当有多个线程进入时，我们就利用该方法，第一个线程进入时，redis 中就有这个key 了，返回了1，如果结果是1，则表示他抢到了锁，那么他去执行业务，然后再删除锁，退出锁逻辑，没有抢到锁的哥们，等待一定时间后重试即可

### 版本一：

这里做了防止锁误删处理

防止线程一删锁前出现堵塞，==锁超时自动释放==；线程二拿到锁，还没到线程二自己删除锁，线程一的删除操作执行了，删除了线程二的锁。

加锁时：加入了获取线程标识，删锁时：判断线程标识是否属于自己的操作。

**锁的基本接口**

~~~java
package com.hmdp.utils;

public interface ILock {
    /**
     *获取锁
     * @param timeoutSec 锁持有的超时时间
     * @return true 成功 false失败
     */
    boolean tryLock(long timeoutSec);

    /**
     * 释放锁
     */
    void unLock();

}

~~~

**SimpleRedisLock**

利用setnx方法进行加锁，同时增加过期时间，防止死锁，此方法可以保证加锁和增加过期时间具有原子性

```java
private static final String KEY_PREFIX="lock:"
@Override
public boolean tryLock(long timeoutSec) {
    // 获取线程标示
    long threadId = Thread.currentThread().getId();
    // 获取锁
    Boolean success = stringRedisTemplate.opsForValue()
            .setIfAbsent(KEY_PREFIX + name, threadId + "", timeoutSec, TimeUnit.SECONDS);
    return Boolean.TRUE.equals(success);
}
```

* 释放锁逻辑

SimpleRedisLock

释放锁，防止删除别人的锁

```java
public void unlock() {
    //通过del删除锁
    stringRedisTemplate.delete(KEY_PREFIX + name);
}
```

### 问题一：此时分布式锁的原子性问题还没解决

线程1判断同一线程标识后，==开始删除锁时卡顿==，自己的锁被超时释放，同一用户的另一线程2拿到锁开始执行业务，此时线程1由于==已近判断过了直接删除锁==，同一用户的线程3拿到锁，和线程2并发了

关键：==判断锁标识和释放锁是两个动作，两个动作之间发生阻塞导致了并发==

![image-20261019121254448](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019121254448.png)

### 版本二：

**用Lua脚本解决原子性问题**。

这里将删除锁逻辑写成lua脚本保证原子性。

~~~lua
-- 这里的 KEYS[1] 就是锁的key，这里的ARGV[1] 就是当前线程标示
-- 获取锁中的标示，判断是否与当前线程标示一致
if (redis.call('GET', KEYS[1]) == ARGV[1]) then
  -- 一致，则删除锁
  return redis.call('DEL', KEYS[1])
end
-- 不一致，则直接返回
return 0
~~~

SimpleRedisLock中unlock()调用lua脚本

~~~java
public class SimpleRedisLock implements ILock{
    private StringRedisTemplate stringRedisTemplate;
    private String name;
    public SimpleRedisLock(String name,StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.name=name;
    }
    private static final String KEY_PREFIX="lock:";
    private static final String ID_PREFIX= UUID.randomUUID().toString();
    /**
     *事先引入lua脚本以节省开支
     */
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static{
        UNLOCK_SCRIPT=new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }
    @Override
    public boolean tryLock(long timeoutSec) {
        //获取线程标识
        String threadId = ID_PREFIX+Thread.currentThread().getId();
        //获取锁
        //value值要求是String类型，threadId是long类型，threadId+""转为String类型
        Boolean isLock = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId , timeoutSec, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(isLock);
    }
    @Override
    public void unLock() {
        //lua脚本实现原子性 判断删除操作
        stringRedisTemplate.execute(UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX + Thread.currentThread().getId());
    }

//    @Override
//    public void unLock() {
//        //删锁时判断锁是否是自己的
//        //获取当下线程标识
//        String threadId = ID_PREFIX + Thread.currentThread().getId();
//        //获取当下锁标识
//        String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
//        //判断标识是否一致
//        if (threadId.equals(id)) {
//            //释放锁
//            stringRedisTemplate.delete(KEY_PREFIX+name);
//        }
//    }
}

~~~



## 5、分布式锁-Redisson

### 5.1分布式锁解决四个问题

- 可重入（Hash，默认）
- 可重试（pub/sub，传入waitTime）
- 可续期（看门狗,不传入leaseTime启动看门狗，传入leaseTime不启动看门狗）
- 主从一致（RedLock）

#### 关键：

trylock的四个重载方法

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

### 可重入

重点在lua脚本使用**hash**结构来判**断线程标识**，**记录重入次数**

~~~java
<T> RFuture<T> tryLockInnerAsync(
        long waitTime,
        long leaseTime,
        TimeUnit unit,
        long threadId,
        RedisStrictCommand<T> command) {

    // 1. 记录租期（毫秒），供看门狗续期使用
    this.internalLockLeaseTime = unit.toMillis(leaseTime);

    // 2. 执行加锁 Lua，返回异步结果
    return this.<T>evalWriteAsync(
            this.getName(),                              // Redis key（锁名）
            LongCodec.INSTANCE,                          // 结果编解码器
            command,                                     // 返回类型解码方式
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                "return nil; " +
            "end; " +
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                "return nil; " +
            "end; " +
            "return redis.call('pttl', KEYS[1]);",           // Lua 脚本
            Collections.singletonList(this.getName()),       // KEYS[1]
            this.internalLockLeaseTime,                      // ARGV[1]
            this.getLockName(threadId)                       // ARGV[2]
    );
}
~~~

### 可重试

~~~java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    // 等待时间换算成毫秒，作为剩余时间预算
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();

    // ① 第一次尝试加锁：null 成功，数字失败（剩余 TTL）
    Long ttl = this.tryAcquire(waitTime, leaseTime, unit, threadId);
    if (ttl == null) {
        return true;
    }

    // ② 扣掉耗时，预算耗尽直接失败（waitTime=-1 走这里）
    time -= System.currentTimeMillis() - current;
    if (time <= 0L) {
        this.acquireFailed(waitTime, unit, threadId);
        return false;
    }

    // ③ 订阅锁释放频道，失败时靠它被唤醒而非轮询
    current = System.currentTimeMillis();
    RFuture<RedissonLockEntry> subscribeFuture = this.subscribe(threadId);
    if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {
        // 订阅建立超时：取消订阅并失败返回
        if (!subscribeFuture.cancel(false)) {
            subscribeFuture.onComplete((res, e) -> {
                if (e == null) {
                    this.unsubscribe(subscribeFuture, threadId);
                }
            });
        }
        this.acquireFailed(waitTime, unit, threadId);
        return false;
    }

    // ④ 重试循环：尝试 → 等信号 → 再尝试
    try {
        time -= System.currentTimeMillis() - current;
        if (time > 0L) {
            do {
                // 再尝试一次
                ttl = this.tryAcquire(waitTime, leaseTime, unit, threadId);
                if (ttl == null) {
                    return true;
                }

                // 扣时间，预算耗尽则失败
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0L) {
                    this.acquireFailed(waitTime, unit, threadId);
                    return false;
                }

                // 在 Semaphore 上限时等信号：等锁释放通知或对方 TTL 到期
                currentTime = System.currentTimeMillis();
                if (ttl >= 0L && ttl < time) {
                    ((RedissonLockEntry) subscribeFuture.getNow()).getLatch()
                            .tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    ((RedissonLockEntry) subscribeFuture.getNow()).getLatch()
                            .tryAcquire(time, TimeUnit.MILLISECONDS);
                }
                time -= System.currentTimeMillis() - currentTime;
            } while (time > 0L);

            // 循环退出，预算耗尽
            this.acquireFailed(waitTime, unit, threadId);
            return false;
        }

        this.acquireFailed(waitTime, unit, threadId);
        return false;
    } finally {
        // 无论成功失败都取消订阅，防泄漏
        this.unsubscribe(subscribeFuture, threadId);
    }
}
~~~

框架注释要点（四段 + 一收尾）

| 段      | 注释                                                  |
| :------ | :---------------------------------------------------- |
| ①       | 第一次尝试，`null` 成功、数字失败                     |
| ②       | 扣耗时，预算耗尽直接失败（`waitTime=-1` 走此）        |
| ③       | 订阅锁释放频道，失败后靠通知唤醒                      |
| ④       | `do-while` 重试：尝试 → Semaphore 限时等信号 → 再尝试 |
| finally | 取消订阅，防泄漏                                      |

### 可续期（看门狗）

**是否传入leaseTime分支**

~~~java
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    if (leaseTime != -1L) {
        // 传了租期：直接执行加锁 Lua，不续期
        return this.<Long>tryLockInnerAsync(
                waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        // 没传租期：用 watchdogTimeout 加锁，成功后启动看门狗
        RFuture<Long> ttlRemainingFuture = this.<Long>tryLockInnerAsync(
                waitTime,
                this.commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
                TimeUnit.MILLISECONDS,
                threadId,
                RedisCommands.EVAL_LONG);

        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e == null) {
                if (ttlRemaining == null) {          // null = 加锁成功
                    this.scheduleExpirationRenewal(threadId);   // 启动看门狗
                }
            }
        });
        return ttlRemainingFuture;
    }
}
~~~

**续期具体实现**

~~~java
private void renewExpiration() {
    // 取出续期登记项；为空说明锁已释放/已停止续期，直接退出
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(this.getEntryName());
    if (ee != null) {
        // 延迟 internalLockLeaseTime/3（默认 10s）后执行续期
        Timeout task = this.commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            public void run(Timeout timeout) throws Exception {
                // 触发时再查一次，防止延迟期间锁已被释放
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(this.getEntryName());
                if (ent != null) {
                    Long threadId = ent.getFirstThreadId();
                    if (threadId != null) {
                        // 执行续期 Lua：重置锁 TTL
                        RFuture<Boolean> future = renewExpirationAsync(threadId);
                        future.onComplete((res, e) -> {
                            if (e != null) {
                                // 续期异常，记录日志，终止续期链
                                log.error("Can't update lock " + getName() + " expiration", e);
                            } else {
                                if (res) {
                                    // 续期成功 → 递归安排下一次，形成循环
                                    renewExpiration();
                                }
                                // res == false：锁已非己有，不递归，链终止
                            }
                        });
                    }
                }
            }
        }, this.internalLockLeaseTime / 3L, TimeUnit.MILLISECONDS);

        // 保存任务句柄，供 unlock 时 cancel()
        ee.setTimeout(task);
    }
}
~~~

#### 核心逻辑

1. **延迟 10s** 执行续期（TTL 的 1/3，留出余量）。
2. **续期成功 → 递归**，每 10s 续一次，形成看门狗循环。
3. **终止条件**：异常、锁已非己有、或 `unlock` 时 `cancel + MAP 移除`（任务触发时 `MAP.get` 为空）。
4. **两处判空**：入口快速退出 + 触发时防延迟期间释放。

### 主从一致

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

![image-20261019170319120](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261019170319120.png)

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

## 6、秒杀优化

### 需求：

当前校验一人一单、库存超卖、减库存、下订单...都是Mysql在做，而且是串行执行。这样很耗费时间。

为解决这种问题，可以使用**削峰解耦**的思想。把校验一人一单、库存超卖这些操作由redis判断，判断符合资格后先**返回下单成功的数据给前端**。然后**真正落库的操作交给Mysql慢慢做**。这样在用户端感受到的响应速度会更快，体验更好。

### 解决：

先讲一下**教程的解决思路**是通过lua脚本

- 校验一人一单：使用SADD，以 ‘seckill:order:’ +voucher_id为key，user_id为value。用`SISMEMBER orderKey userId`判断是否存在，如果存在则该用户下过一次单,返回 `2` ;不存在则没下过，执行下单（保存用户）`sadd orderKey userId`
- 校验库存是否充足：以 'seckill:stock:' +voucher_id为key，提前预热这个key,将value设置为100（准备发放的优惠卷数量）。判断`get stockKey`<=0，条件成立返回1，代表库存不足。反之则 `incrby stockKey -1`扣减库存。

| 分类         | 检查/操作          | Key（类型）                           | 命令               | 判断条件      | 结果               |
| :----------- | :----------------- | :------------------------------------ | :----------------- | :------------ | :----------------- |
| **扣减库存** | 校验库存是否充足   | `seckill:stock:{voucherId}`（String） | `GET`              | `<= 0`        | 库存不足，返回 `1` |
| **扣减库存** | 扣减库存           | `seckill:stock:{voucherId}`           | `INCRBY -1`        | `> 0` 时执行  | 库存 -1            |
| **一人一单** | 校验用户是否已下单 | `seckill:order:{voucherId}`（Set）    | `SISMEMBER userId` | `== 1`        | 重复下单，返回 `2` |
| **一人一单** | 记录用户已下单     | `seckill:order:{voucherId}`           | `SADD userId`      | `== 0` 时执行 | 用户加入集合       |

首先这样可以，但是校验一人一单完全可以用SETNX来操作，setnx天然带有互斥操作，如果原来存在则不执行。
显然更加清爽。
所以要灵活使用，不能模板化，教程这样教应该是为了更好的学习lua，但如果学死了，就会感到困惑（学习RocketMQ事用的setnx,当时脑子乱了。仔细思考发现两者都可）

Set类型，不带过期时间（Set是key加上集合）

~~~java
@Resource
private StringRedisTemplate stringRedisTemplate;

public boolean tryOrder(Long voucherId, Long userId) {
    String orderKey = "seckill:order:" + voucherId;

    // SADD 返回 1 = 新加入（首次下单），0 = 已存在（重复）
    Long added = stringRedisTemplate.opsForSet().add(orderKey, userId.toString());
    return added != null && added == 1L;
}
~~~

或String类型，**可以**带**过期时间**（String 是key加上单个String）

~~~java
String key = "seckill:order:" + voucherId + ":" + userId;   // 券+用户拼 key
Boolean ok = stringRedisTemplate.opsForValue()
        .setIfAbsent(key, "1", Duration.ofDays(1));         // 带过期，原子
~~~



**教程解决流程**

lua

~~~lua
-- 1.参数列表
-- 1.1.优惠券id
local voucherId = ARGV[1]
-- 1.2.用户id
local userId = ARGV[2]
--使用Redis消息队列时才用到，此时没有定义传入订单id的逻辑，仔细想想现阶段也用不到这个
-- 1.3.订单id
local orderId = ARGV[3]

-- 2.数据key
-- 2.1.库存key
local stockKey = 'seckill:stock:' .. voucherId
-- 2.2.订单key
local orderKey = 'seckill:order:' .. voucherId

-- 3.脚本业务
-- 3.1.判断库存是否充足 get stockKey
if(tonumber(redis.call('get', stockKey)) <= 0) then
    -- 3.2.库存不足，返回1
    return 1
end
-- 3.2.判断用户是否下单 SISMEMBER orderKey userId
if(redis.call('sismember', orderKey, userId) == 1) then
    -- 3.3.存在，说明是重复下单，返回2
    return 2
end
-- 3.4.扣库存 incrby stockKey -1
redis.call('incrby', stockKey, -1)
-- 3.5.下单（保存用户）sadd orderKey userId
redis.call('sadd', orderKey, userId)
--3.6在Redis消息队列中才会用到，现阶段没有用到
-- 3.6.发送消息到队列中， XADD stream.orders * k1 v1 k2 v2 ...
redis.call('xadd', 'stream.orders', '*', 'userId', userId, 'voucherId', voucherId, 'id', orderId)
return 0
~~~

业务处:

#### 基于阻塞队列

lua判断资格后，业务调用lua脚本获取验证结果。

0：成功->创建订单实体类先返回前端->订单信息放入阻塞队列->开启异步线程获取阻塞队列->兜底验证一人一单以及库存超卖（幂等操作，防止网络波动导致重复消费）->落库扣减库存、创建订单

1、2：直接返回失败。

## 7、Redis消息队列

**tip：**

**Stream消息队列**是==拉模式==

具体表现形式为：**生产端**只负责生产消息推送到消息队列，**消费方**主动**声明自己是那个消费者组的那个消费者**然后到消息队列中拉取数据消费。

**RocketMQ**消息队列**在客户端 API 层面表现为"推模式"（Push Consumer）**

具体表现形式为：

**生产者：**

1.创建消息生产者producer，并制定生产者组名

2.指定Nameserver地址

3.启动producer

4.创建消息对象，==指定主题Topic、Tag和消息体等==

5.发送消息

6.关闭生产者producer

**消费者：**

1.创建消费者consumer，制定消费者组名

2.指定Nameserver地址

3.==监听订阅主题Topic和Tag等==

4.处理消息

5.启动消费者consumer

**RocketMQ底层实现是"长轮询拉模式"（Pull + Long Polling）。** 也就是说：**对用户是推，对底层是拉。**

对大部分的RocketMQ场景直接理解成==推模式==即可。

### 7.1Redis的Stream结构作为消息队列，实现异步秒杀下单

#### 需求：

* 新增秒杀优惠券的同时，将优惠券信息保存到Redis中

* 基于Lua脚本，判断秒杀库存、一人一单，决定用户是否抢购成功

* 如果抢购成功，将优惠券id和用户id封装后存入阻塞队列

* 开启线程任务，不断从阻塞队列中获取信息，实现异步下单功能



lua脚本中,使用`XADD stream.orders * k1 v1 k2 v2 ...`**向名为 `stream.orders` 的 Stream（消息队列）追加**待消费的消息。

~~~lua
-- 3.6.发送消息到队列中， XADD stream.orders * k1 v1 k2 v2 ...
redis.call(          -- 在 Lua 脚本里执行一条 Redis 命令（原子，和脚本其他命令一起）
    'xadd',          -- Redis 命令：向 Stream 追加一条消息
    'stream.orders', -- Stream 的 key 名（消息队列的名字）
    '*',             -- 消息 ID 由 Redis 自动生成（* = 自动生成，格式 时间戳-序号）
    'userId',        -- 字段名 1
    userId,          -- 字段值 1（用户 id）
    'voucherId',     -- 字段名 2
    voucherId,       -- 字段值 2（优惠券 id）
    'id',            -- 字段名 3
    orderId          -- 字段值 3（订单 id）
)
return 0
~~~

开启线程任务消费消息队列

执行lua脚本：

~~~java
  public Result sekillVoucher(Long voucherId) {
        //注入ISeckillVoucherService,通过id获取秒杀卷信息
        SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
        //获取代理对象
        proxy = (IVoucherOrderService) AopContext.currentProxy();
        // 1.判断秒杀是否开始或结束
        if (seckillVoucher.getBeginTime().isAfter(LocalDateTime.now())) {
            return Result.fail("秒杀尚未开始");
        }
        if (seckillVoucher.getEndTime().isBefore(LocalDateTime.now())) {
            return Result.fail("秒杀已经结束");
        }

        //ThreadLocal获取用户信息
        Long userId = UserHolder.getUser().getId();
        long orderId = redisIdWorker.nextId("order");
        //1.执行lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(), userId.toString(),String.valueOf(orderId));
        int r = result.intValue();
        //2.判断是否为0
        if (r!=0) {
            //不为0，没有购买资格
            return Result.fail(r==1?"库存不足！":"不能重复下单！");
        }
        //3.为0，有购买资格，异步线程对消息队列中的订单执行落库操作

        //4.返回订单id
        return Result.ok(orderId);

    }
~~~

异步线程消费:

![消息消费流程图_v2](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/%E6%B6%88%E6%81%AF%E6%B6%88%E8%B4%B9%E6%B5%81%E7%A8%8B%E5%9B%BE_v2.png)

~~~java
private class VoucherOrderHandler implements Runnable {

    @Override
    public void run() {
        while (true) {
            try {
                // 1.获取消息队列中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 >
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),// // 我声明：我是 g1 组的 c1
                    StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                    StreamOffset.create("stream.orders", ReadOffset.lastConsumed())
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有消息，继续下一次循环
                    continue;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("stream.orders", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理订单异常", e);
                //处理异常消息
                handlePendingList();
            }
        }
    }

    private void handlePendingList() {
        while (true) {
            try {
                // 1.获取pending-list中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 0
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),
                    StreamReadOptions.empty().count(1),
                    StreamOffset.create("stream.orders", ReadOffset.from("0"))
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有异常消息，结束循环
                    break;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("stream.orders", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理pendding订单异常", e);
                try{
                    Thread.sleep(20);
                }catch(Exception e){
                    e.printStackTrace();
                }
            }
        }
    }
}

~~~

## 8、达人探店

### 8.1点赞功能

初始代码

~~~java
@GetMapping("/like/{id}")
public Result queryBlogLikes(@Pa	thVariable("id") Long id) {
    //修改点赞数量
    blogService.update().setSql("liked = liked +1 ").eq("id",id).update();
    return Result.ok();
}
~~~

#### 问题：

这种方式会导致一个用户无限点赞，明显是不合理的

#### 需求：

* 同一个用户只能点赞一次，再次点击则取消点赞
* 如果当前用户已经点赞，则点赞按钮高亮显示（前端已实现，判断字段Blog类的isLike属性）

#### 解决：

实现步骤：

* 给Blog类中添加一个**isLike字段**，标示是否被当前用户点赞
* 修改点赞功能，利用Redis的**set集**合判断是否点赞过，未点赞过则点赞数+1，已点赞过则点赞数-1
* 修改根据id查询Blog的业务，判断当前登录用户是否点赞过，赋值给isLike字段
* 修改分页查询Blog业务，判断当前登录用户是否点赞过，赋值给isLike字段

注意：这个**isLike**不属于数据库中Blog表字段，使用了`@TableField(exist = false)`标记了。通过判断**redis**中博客用户点赞关系，来确定这个字段是true还是false。

~~~java
@TableField(exist = false)
private Boolean isLike;
~~~

如果数据库中想要设计用户和博客的点赞关系，一个用户可以给多个博客点赞，一个博客可以被多个用户点赞。典型的多对多关系，用户表、博客表已近有了，所以想要建立关系，应该建立一个中间关系表user_blog。点赞时，写入中间关系表；获取点赞状态时，通过隐式或显示（推荐）来查询**具体某个博客被哪些用户点赞。**

显示查询（JOIN,用的内连接）

~~~sql
select u.id
from tb_user u 
join user_blog ub on u.id=ub.user_id 
join tb_blog b on b.id=ub.blog_id
where b.id=？
~~~

隐式查询

~~~sql
select u.id
form tb_user u,user_blog ub,tb_blog b
where u.id=ub.user.id and b.id=ub.blog.id and b.id=？
~~~

#### 言归正传

教程用了redis来缓存用户博客点赞关系，此时采用的是SET这种数据结构。`SISMEMBER`判断点赞关系

key : `blog_id`

value:  `userId`

利用Redis的**set集**合判断是否点赞过

~~~JAVA
/**
     * 判断登录用户是否点赞
     * 若点赞将isLike设置为true
     * @param blog
     */
    private void isBlogLiked(Blog blog) {
        //1.获取登录用户
        UserDTO user = UserHolder.getUser();
        if (user==null) {
            //1.1判断是否登录，若为null证明未登录状态,无需查询是否点赞
            return;
        }
        Long userId = user.getId();
        //2.判断是否点赞
        Boolean isMember = stringRedisTemplate.opsForSet().isMember("blog:liked:"+ blog.getId(),userId.toString());
        //3.若点赞，isLike设置为true
        blog.setIsLike(BooleanUtil.isTrue(isMember));
    }
~~~

未点赞过则点赞数+1，已点赞过则点赞数-1

~~~java
@Override
    public Result likeBlog(Long id){
        // 1.获取登录用户
        Long userId = UserHolder.getUser().getId();
        // 2.判断当前登录用户是否已经点赞
        String key = BLOG_LIKED_KEY + id;
        Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
        if(BooleanUtil.isFalse(isMember)){
             //3.如果未点赞，可以点赞
            //3.1 数据库点赞数+1
            boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
            //3.2 保存用户到Redis的set集合
            if(isSuccess){
                stringRedisTemplate.opsForSet().add(key,userId.toString());
            }
        }else{
             //4.如果已点赞，取消点赞
            //4.1 数据库点赞数-1
            boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
            //4.2 把用户从Redis的set集合移除
            if(isSuccess){
                stringRedisTemplate.opsForSet().remove(key,userId.toString());
            }
        }
~~~

### 8.2点赞排行榜

#### 需求

在探店笔记的详情页面，应该把给该笔记点赞的人显示出来，比如最早点赞的TOP5，形成点赞排行榜：

之前的点赞是放到set集合，但是set集合是不能排序的，所以这个时候，咱们可以采用一个可以排序的set集合，就是**SortedSet**

**SortedSe**t中没有SISMEMBER命令来判断是否存在

使用`ZSCORE key member` : 获取sorted set中的指定元素的score值;若为空则未点赞，非空点过赞

#### 解决

1、likeBlog改用SortedSet

保存用户到Redis的set集合：  zadd key value score。这里的score用**时间戳**，这样就能**按照点赞先后排序**了。

~~~java
  @Override
    public Result likeBlog(Long id) {
        // 1.获取登录用户
        Long userId = UserHolder.getUser().getId();
        // 2.判断当前登录用户是否已经点赞
        String key = BLOG_LIKED_KEY + id;
        Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
        if (score == null) {
            // 3.如果未点赞，可以点赞
            // 3.1.数据库点赞数 + 1
            boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
            // 3.2.保存用户到Redis的set集合  zadd key value score
            if (isSuccess) {
                stringRedisTemplate.opsForZSet().add(key, userId.toString(), System.currentTimeMillis());
            }
        } else {
            // 4.如果已点赞，取消点赞
            // 4.1.数据库点赞数 -1
            boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
            // 4.2.把用户从Redis的set集合移除
            if (isSuccess) {
                stringRedisTemplate.opsForZSet().remove(key, userId.toString());
            }
        }
        return Result.ok();
    }
~~~

2、获取点赞排行榜

ZRANGE key min max：按照score排序后，获取==指定排名范围==内的元素

ZRANGE KEY 0 4 获取点赞前5名的数据

涉及知识点：

- ZRANGE key min max
- stream流
- 方法引用
- .last("ORDER BY FIELD(id," + idStr + ")")，IN返回顺序不保证，用ORDER BY FIELD指定返回顺序

~~~java
public Result queryBlogLikes(Long id) {
    String key = BLOG_LIKED_KEY + id;
    // 1.查询top5的点赞用户 zrange key 0 4
    Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
    if (top5 == null || top5.isEmpty()) {
        return Result.ok(Collections.emptyList());
    }
    // 2.解析出其中的用户id
    List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
    String idStr = StrUtil.join(",", ids);
    // 3.根据用户id查询用户 WHERE id IN ( 5 , 1 ) ORDER BY FIELD(id, 5, 1)
    List<UserDTO> userDTOS = userService.query()
            .in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list()
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    // 4.返回
    return Result.ok(userDTOS);
}
~~~



## 9、好友关注

### 9.1 好友关注-关注和取消关注

#### 需求：

针对用户的操作：可以对用户进行关注和取消关注功能。

#### 解决：

实现思路：

需求：基于该表数据结构，实现两个接口：

* 关注和取关接口
* 判断是否关注的接口

关注是User之间的关系，是**博主**与**用户**的粉丝关系（多对多关系），数据库中有一张tb_follow表来标示：

![image-20261020114511356](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261020114511356.png)

**tip:**学习过**自连接**查询后，其实这种**既当裁判又当参赛选手**的数据关系，完全可以用一张表来标识，tb_user表中直接加个字段follow_user_id。

查询粉丝博主关系时：

隐式：

~~~sql
select u1.user_id,u2.follow_user_id
from tb_user u1 
join tb_user u2 on u1.user_id=u2.follow_user_id
~~~

显式：

~~~sql
select u1.user_id,u2.follow_user_id
from tb_user u1 ,tb_user u2
where u1.user_id=u2.follow_user_id
~~~



**实现两个功能**

1. 获取登录用户的关注状态返回前端（前端：用于标识是否关注）
2. 根据前端返回的关注状态 关注或取关（前端：点击按钮，如果关注了，点击一下取关；如果没关注，点击一下关注）

~~~java
//获取关注状态
@Override
public Result isFollow(Long followUserId) {
        // 1.获取登录用户
        Long userId = UserHolder.getUser().getId();
        // 2.查询是否关注 select count(*) from tb_follow where user_id = ? and follow_user_id = ?
        Integer count = query().eq("user_id", userId).eq("follow_user_id", followUserId).count();
        // 3.判断
        return Result.ok(count > 0);
    }

//根据关注状态 关注或取关
 @Override
    public Result follow(Long followUserId, Boolean isFollow) {
        // 1.获取登录用户
        Long userId = UserHolder.getUser().getId();
        String key = "follows:" + userId;
        // 1.判断到底是关注还是取关
        if (isFollow) {
            // 2.关注，新增数据
            Follow follow = new Follow();
            follow.setUserId(userId);
            follow.setFollowUserId(followUserId);
            boolean isSuccess = save(follow);

        } else {
            // 3.取关，删除 delete from tb_follow where user_id = ? and follow_user_id = ?
            remove(new QueryWrapper<Follow>()
                    .eq("user_id", userId).eq("follow_user_id", followUserId));

        }
        return Result.ok();
    }
~~~

### 9.2好友关注-共同关注

#### 需求：

进入博主页面时，可以获取到博主和自己共同关注了哪些好友

#### 解决：

这里优先考虑用**redis**中的**SET**来存储博主被哪些人关注。

单纯使用mysql可以做到获取共同关注吗？当然可以，但是你肯定要涉及多张表来建立建立连接关系，tb_follow就可以作为中间表

当你想要实现共同关注时，就要注意到一个博主可以被多个粉丝关注，一个粉丝可以关注多个博主，**典型的多对多关系**。

1、**用户表可以代表博主表和粉丝表**，**tb_follow表代表中间表**。

2、确认表关系后，用mysql中的**内连接（[INNER] JOIN）**求交集即可。

具体JOIN怎么做：

注意：你**直接用博主表和粉丝表求交集是求不到的**，他俩没有直接关系。需要通过中间表建立联系后求。

一、mysql层用sql直接获取(给其中一种实现方法)

~~~sql
-- 共同关注（A 和 B 都关注的人）
SELECT u.id, u.nickname
FROM tb_follow f1
JOIN tb_follow f2 ON f1.follow_user_id = f2.follow_user_id
JOIN tb_user u     ON u.id = f1.follow_user_id
WHERE f1.user_id = ?   -- A 的 id
  AND f2.user_id = ?;  -- B 的 id

~~~

二、mysql层用sql分别获取A关注了哪些人，B关注了哪些人。然后业务层自己手动遍历求交集

~~~sql
-- A 的关注列表
SELECT u2.id
FROM tb_user u1
JOIN tb_follow f
  ON u1.id = f.user_id
JOIN tb_user u2
  ON u2.id = f.follow_user_id
WHERE u1.id = ?;


-- B 的关注列表
SELECT u1.id
FROM tb_user u1
JOIN tb_follow f
  ON u1.id = f.user_id
JOIN tb_user u2
  ON u2.id = f.follow_user_id
WHERE u2.id = ?;
~~~

业务层:有很多方法，比如用stream流遍历A，然后对每个A求`B.contains(A)`,即B中是否含有A中的某个值。
~~~java
List<Integer> result = a.stream()
    .filter(b::contains)          // 对 a 的每个元素 x，判断 b.contains(x)
    .collect(Collectors.toList());
~~~

**总结**：Mysql中这种多对多关系求交集真的很麻烦，而且join太多了性能很差，这也是为什么课程用redis的数据结构来求交集。

#### 言归正传

1、用redis的**SET**数据结构存储 **博主-所有关注者**的数据,博主id为key,关注者id为value。每次**关注或者取关**后将当下用户的数据**存入redis**中。

2、获取共同关注时，用redis中的 SINTER key1 key2 ... ：求key1与key2的交集。只需要通过key就能很快获取共同的交集比mysql要方便很多。

tip:第一步关注或取关的操作，按理说应该在redis和mysql中都要落库才对，这里**课程为了方便直接将落库mysql改成了落库redis**。

**关注或取关时：**

~~~java
@Override
public Result follow(Long followUserId, Boolean isFollow) {
    // 1.获取登录用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 1.判断到底是关注还是取关
    if (isFollow) {
        // 2.关注，新增数据
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        boolean isSuccess = save(follow);
        if (isSuccess) {
            // 把关注用户的id，放入redis的set集合 sadd userId followerUserId
            stringRedisTemplate.opsForSet().add(key, followUserId.toString());
        }
    } else {
        // 3.取关，删除 delete from tb_follow where user_id = ? and follow_user_id = ?
        boolean isSuccess = remove(new QueryWrapper<Follow>()
                .eq("user_id", userId).eq("follow_user_id", followUserId));
        if (isSuccess) {
            // 把关注用户的id从Redis集合中移除
            stringRedisTemplate.opsForSet().remove(key, followUserId.toString());
        }
    }
    return Result.ok();
}
~~~

**获取共同关注:**

~~~java
@Override
public Result followCommons(Long id) {
    // 1.获取当前用户
    Long userId = UserHolder.getUser().getId();
    String key = "follows:" + userId;
    // 2.求交集
    String key2 = "follows:" + id;
    //SINTRE key key2
    Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key, key2);
    if (intersect == null || intersect.isEmpty()) {
        // 无交集
        return Result.ok(Collections.emptyList());
    }
    // 3.解析id集合
    List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
    // 4.查询用户
    List<UserDTO> users = userService.listByIds(ids)
            .stream()
            .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
            .collect(Collectors.toList());
    return Result.ok(users);
}
~~~

### 9.3好友关注-Feed流

#### 概念：这里看实战教程更清楚

当我们关注了用户后，这个用户发了动态，那么我们应该把这些数据推送给用户，这个需求，其实我们又把他叫做Feed流，关注推送也叫做Feed流，直译为投喂。为用户持续的提供“沉浸式”的体验，通过无限下拉刷新获取新的信息。

Feed流的实现有两种模式：

Feed流产品有两种常见模式：
Timeline：不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

* 优点：信息全面，不会有缺失。并且实现也相对简单
* 缺点：信息噪音较多，用户不一定感兴趣，内容获取效率低

智能排序：利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户

* 优点：投喂用户感兴趣信息，用户粘度很高，容易沉迷
* 缺点：如果算法不精准，可能起到反作用
  本例中的个人页面，是基于关注的好友来做Feed流，因此采用Timeline的模式。该模式的实现方案有三种：

我们本次针对好友的操作，采用的就是Timeline的方式，只需要拿到我们关注用户的信息，然后按照时间排序即可

，因此采用Timeline的模式。该模式的实现方案有三种：

* 拉模式
* 推模式
* 推拉结合

#### 拉模式：

博主将博客放进自己的发件箱，用户获取的时候现读取到自己的收件箱

#### 推模式：

博主发布博客的时候直接推送到用户收件箱

#### 推拉结合：

大V角度：先写博客进发件箱，对活跃粉丝用推模式，对普通用户用拉模式

普通用户角度：直接推模式

### 9.4好友关注-推送到粉丝收件箱（推模式）

#### 需求：

* 修改新增探店笔记的业务，在保存blog到数据库的同时，推送到粉丝的收件箱
* 收件箱满足可以根据时间戳排序，选择用Redis的数据结构SortedSet实现
* 查询收件箱数据时，可以实现分页查询

#### 思路：

**推送内容**

这里推送时，不会将整个博客内容推送到粉丝收件箱中，那样太消耗内存了，选择将博客id推送过去即可，粉丝看博客时按照推送过来的博客id查即可

**key和value怎么选**

推模式（当下）：一个粉丝一个粉丝箱中可以收到不同博主的博客，所以以**粉丝id为key,博客id为value，时间戳作为score**

拉模式：博主有发件箱,一个博主对应一个发件箱可以发不同的博客，所以**发件箱中博主id为key，博客id为value,时间戳为score**;一个粉丝一个收件箱，读取信息时会从关注的不同博主中的收件箱信息全部拉过来，所以收件箱中**粉丝id为key,==博主id为value==**，查询时先根据博主id查询对应的博主发件箱，再从发件箱中拉去博客。

#### 解决：

保存博客时，查询博主粉丝，将笔记id推送给粉丝

~~~java
public Result saveBlog(Blog blog) {
    // 1.获取登录用户
    UserDTO user = UserHolder.getUser();
    blog.setUserId(user.getId());
    // 2.保存探店笔记
    boolean isSuccess = save(blog);
    if(!isSuccess){
        return Result.fail("新增笔记失败!");
    }
    // 3.查询笔记作者的所有粉丝 select * from tb_follow where follow_user_id = ?
    List<Follow> follows = followService.query().eq("follow_user_id", user.getId()).list();
    // 4.推送笔记id给所有粉丝
    for (Follow follow : follows) {
        // 4.1.获取粉丝id
        Long userId = follow.getUserId();
        // 4.2.推送
        String key = FEED_KEY + userId;
        stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
    }
    // 5.返回id
    return Result.ok(blog.getId());
}
~~~

### 9.5好友关注-实现滚动分页查询

#### 需求：

首先我们业务需要的查询逻辑是，按照**时间戳大小从大往小查**，也就是说最新的会放在最前面。

这就导致，数据的角标会从头开始往下挤。

如果还按照原来的`ZREVGEBYSCORE key min max  [LIMIT offset count]`,说明一下这个命令，如果只带`min max`那就会返回这个范围内的所有数据，现实业务大概率不会这样，都会做分页查询，会带上`LIMIT offset count`表示跳过前offset位（不包括该位）查count个数据`(offset,offset+count] `。==这里还要注意，从那个起点开始跳过offset位呢？起点是max。==所以offset代表的是偏移量，==起点加偏移量才是下一次的起点==

这里的起点max始终保持最大值，然后由offset来计算出每次的起点，**此时offset虽然是偏移量，但它代表了起点**。

因为这里计算起点的方式是传统的`long offset = (long) (page - 1) * size;`,代表每次以固定的max起点来计算偏移量，但是步数固定，数据本身从max开始往下挤，导致会有重复的数据被读取。

**固定索引分页查询**

![ZSET分页重复读取问题](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/ZSET%E5%88%86%E9%A1%B5%E9%87%8D%E5%A4%8D%E8%AF%BB%E5%8F%96%E9%97%AE%E9%A2%98.gif)

~~~java
@Service
public class ZSetPageService {

    @Autowired
    private StringRedisTemplate redis;

    /**
     * 按 score 降序分页
     * @param key    zset key
     * @param min    最小 score（含）
     * @param max    最大 score（含）
     * @param page   页码，从 1 开始
     * @param size   每页条数
     */
    public Set<ZSetOperations.TypedTuple<String>> pageByScoreDesc(
            String key, double min, double max, int page, int size) {
        long offset = (long) (page - 1) * size;
        return redis.opsForZSet()
                .reverseRangeByScoreWithScores(key, min, max, offset, size);
    }

    // 升序版
    public Set<ZSetOperations.TypedTuple<String>> pageByScoreAsc(
            String key, double min, double max, int page, int size) {
        long offset = (long) (page - 1) * size;
        return redis.opsForZSet()
                .rangeByScoreWithScores(key, min, max, offset, size);
    }
}
~~~

**滚动分页查询**

如何查询这种向头插导致顺序变化的场景，且保证不重复查呢？

这里关键是从源头出发，传统分页查询的起点一直是以第一个位置为起点，**以第一个位置为基准来计算偏移量得到每次查询的起点**。

#### 解决：

第一步：此时我不以第一个位置为起点了，第一次查询后得到查询到的最后一个位置，以这个位置为“起点”minTime返回前端用于定义下一次的max，此时按理说将偏移量offset设为1即可，因为**设为1即代表，跳过该起点本身，从起点的下一个开始查。**

#### 解决：

第二步：但是，有这么一种情况，本身我是以score来标定位置的。如果出现**同score**的情况，此时我返回score来表示最后一个位置，但我**下次查询时会以第一个出现改score的位置为起点，导致后面同分的score又被查了一遍**。

 ① **同分边界**：`max` 会包含该分值的第一条记录

场景：8 条博文，`b8`、`b7` 都是 **109 分**，每页 2 条。

- 第 1 页 `max=+inf, offset=0` → 返回 **b8、b7**，`minTime=109`、`os=2`；
- 如果只把 score 当作位置，下一次查询传入 `max=109, offset=0` 时，扫描会从 109 的第一条（b8）重新开始，于是 **b8、b7 又被返回一遍**；
- 正确做法是带上 `offset = os = 2`，跳过这两条，返回 **b6、b5**。

![同分场景-边界重复](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/%E5%90%8C%E5%88%86%E5%9C%BA%E6%99%AF-%E8%BE%B9%E7%95%8C%E9%87%8D%E5%A4%8D.gif)

为了解决这个问题，需要使用os（计数器）来标记本次查询中，**最后的位置**分数出现重复时的score的数量（默认没有重复时数量为1），重复几个，下次就在此基础上偏移几。**os=offset**即可

~~~java
//3.2 获取 博文id 最小时间 偏移量
        //最小时间
        long minTime = 0;
        //偏移量 （最小时间出现的次数）
        Integer os = 1;
        //博文id列表 （typedTuples.size()指定列表大小保持typedTuples长度一致）
        ArrayList<Long> ids = new ArrayList(typedTuples.size());
        for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
            //博文id
            ids.add(Long.valueOf(typedTuple.getValue()));
            //score(时间戳) longValue() 转为long类型
            long time = typedTuple.getScore().longValue();
            if (time == minTime) {
                os++;
            } else {
                minTime = time;
            }
        }
	os=offset;//这一步需优化
~~~

#### 解决：

第三步：又但是，有这么一种极度极端的情况，比如分数分布是：105 105|105 105|105 105,每次查两个。按照步骤二的思路，第一次查询，offset=2，minTime赋值给下一次的max；下一次查第二组数据，没毛病；第二次查询坏了，由于此时的minTime依旧等于最开始的第一组的时间max相同,此时又查到第一组的数据了。

② 极端同分：整页同分时，`offset` 必须累加

场景：6 条博文分数**全是 105**，`105 105 | 105 105 | 105 105`，每页 2 条。

- 第 1 页 → b6、b5，`max=105, offset=2`；
- 第 2 页 → b4、b3，此时 `minTime == max`，`os=2`；
- 如果第 3 页仍用 `offset=2`，查询会退回第 2 页的起点，于是 **b4、b3 整页重复**；
- 修复方法是 `offset = os + 上一页 offset = 2 + 2 = 4`，第 3 页返回 **b2、b1**；下一次 `2 + 4 = 6`，第 4 页返回空，滚动结束。

![极端同分-整页重复](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/%E6%9E%81%E7%AB%AF%E5%90%8C%E5%88%86-%E6%95%B4%E9%A1%B5%E9%87%8D%E5%A4%8D.gif)

为解决这个问题，要做极端场景的判断。如果minTime==本次的max(即上次的minTime)，就把本次计算出的os和上一次传进来的偏移量相加即可

```java
os = max == minTime ? os + offset : os;
```

![ZSET滚动分页流程](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/ZSET%E6%BB%9A%E5%8A%A8%E5%88%86%E9%A1%B5%E6%B5%81%E7%A8%8B.gif)

~~~java
public Result queryBlogOfFollow(Long max, Integer offset) {
        //1.获取用户id
        Long userId = UserHolder.getUser().getId();
        String key = FEED_KEY + userId;
        //2.根据用户id组成的key查询 value(博文id) score(时间戳)
        //ZREVRANGEBYSCORE key Max Min LIMIT offset count (offset:下一此查询从新开头的第几个开始 count:一次查几条)
        Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet()
                .reverseRangeByScoreWithScores(key, 0, max, offset, 2);
        //3.解析博文id 最小时间戳minTime 偏移量 os
        //3.1非空判断
        if (typedTuples == null || typedTuples.isEmpty()) {
            //空：直接返回
            return Result.ok();
        }
        //3.2 获取 博文id 最小时间 偏移量
        //最小时间
        long minTime = 0;
        //偏移量 （最小时间出现的次数）
        Integer os = 1;
        //博文id列表 （typedTuples.size()指定列表大小保持typedTuples长度一致）
        ArrayList<Long> ids = new ArrayList(typedTuples.size());
        for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
            //博文id
            ids.add(Long.valueOf(typedTuple.getValue()));
            //score(时间戳) longValue() 转为long类型
            long time = typedTuple.getScore().longValue();
            if (time == minTime) {//time要么递减要么不变
                os++;
            } else {//如果不等于一定是time变小了
                minTime = time;
            }
        }
        //解决同分极端场景 5 5 | 5 5 | 5，此时max=minTime，导致查到重复的数据
        //如果minTime==max，说明最大的时间和计算出来的下次要查的最大时间（即minTime）是相同的，此时如果os还是2的话，就会导致重新查到第二组5 5 ，就这样一直重复查到
        //解决办法就是如果出现minTime==max，就把本次计算出来的下次偏移量：“os”和本次传进来的偏移量：“offset”相加即可查到第三组5
        os = max == minTime ? os + offset : os;
        //4.根据id查blog
        String idStr = StrUtil.join(",", ids);
        List<Blog> blogs = query()
                .in("id", ids)
                .last("ORDER BY FIELD(id," + idStr + ")")
                .list();
        //4.1 blog中添加 博客对应的用户信息 博客是否被点赞信息
        for (Blog blog : blogs) {
            //添加博客对应的用户信息
            queryBlogUser(blog);
            //添加点赞信息
            isBlogLiked(blog);
        }
        //5.封装并返回
        ScrollResult scrollResult = new ScrollResult();
        scrollResult.setList(blogs);
        scrollResult.setMinTime(minTime);
        scrollResult.setOffset(os);
        return Result.ok(scrollResult);
    }
~~~

## 10、附近商铺

### Redis地理位置类数据结构-GEO

### 10.1 GEO基本用法

* GEOADD：添加一个地理空间信息，包含：经度（longitude）、纬度（latitude）、值（member）
* GEODIST：计算指定的两个点之间的距离并返回
* GEOPOS：返回指定member的坐标。
* GEOSEARCH：在指定**范围内**搜索member，并按照与指定点之间的距离**排序后返回**。范围可以是圆形或矩形。6.2.新功能
* GEOSEARCHSTORE：与GEOSEARCH功能一致，不过可以把结果存储到一个指定的key。 6.2.新功能

**举个例子**

下面用一套**完整的城市数据**，把 `GEOADD / GEODIST / GEOPOS / GEOSEARCH / GEOSEARCHSTORE` 串起来演示。

#### 一、准备数据：GEOADD

```java
# 语法：GEOADD key 经度 纬度 member [经度 纬度 member ...]
GEOADD cities 116.40 39.90 "beijing"
GEOADD cities 121.47 31.23 "shanghai"
GEOADD cities 113.26 23.13 "guangzhou"
GEOADD cities 114.06 22.54 "shenzhen"
GEOADD cities 117.20 39.13 "tianjin"
GEOADD cities 108.94 34.34 "xian"
```

- 经度在前（-180~180），纬度在后（-85.05~85.05），**顺序不能反**
- 一个 key 下可以有很多 member，底层是 ZSet

```java
TYPE cities
# zset
ZCARD cities
# 6
```

#### 二、查坐标：GEOPOS

```java
# 语法：GEOPOS key member [member ...]
GEOPOS cities beijing shanghai
```

返回：

```java
1) 1) "116.39999896287918091"
   2) "39.90000009167092543"
2) 1) "121.47000110149383545"
   2) "31.22999903975783554"
```



- 返回的是**数组**，每个 member 对应一组 `[经度, 纬度]`
- 精度有浮点误差（正常现象）
- member 不存在时对应位置返回 `nil`

#### 三、算两点距离：GEODIST

```java
# 语法：GEODIST key member1 member2 [unit]
# unit: m（默认）/ km / mi（英里）/ ft（英尺）
GEODIST cities beijing tianjin
# "111.65..."   默认单位米

GEODIST cities beijing tianjin km
# "111.65..."   km

GEODIST cities beijing shanghai km
# "1067.5..."
```



- 任意 member 不存在 → 返回 `nil`
- 距离是**球面大圆距离**（Haversine），不是直线平面距离

#### 四、范围搜索：GEOSEARCH（Redis 6.2+）

##### 1. 圆形范围，按距离升序

```java
# 以北京为中心，1500km 半径内，按距离从近到远
GEOSEARCH cities FROMMEMBER beijing BYRADIUS 1500 km ASC
```

返回：

```text
1) "beijing"
2) "tianjin"
3) "shanghai"
```

##### 2. 带距离和坐标

```java
GEOSEARCH cities FROMMEMBER beijing BYRADIUS 1500 km ASC
    WITHDIST WITHCOORD
```

返回：

```java
1) 1) "beijing"
   2) "0.0000"                    ← 距离
   3) 1) "116.39999896287918091"  ← 经度
      2) "39.90000009167092543"   ← 纬度
2) 1) "tianjin"
   2) "111.6547"
   3) 1) "117.19999879598617554"
      2) "39.12999974070185743"
3) 1) "shanghai"
   2) "1067.5482"
   3) 1) "121.47000110149383545"
      2) "31.22999903975783554"
```



##### 3. 降序 + 限制数量

```java
# 从远到近，只取 2 个
GEOSEARCH cities FROMMEMBER beijing BYRADIUS 1500 km DESC COUNT 2
# → shanghai, tianjin
```



##### 4. 以坐标为中心（不用 member）

```java
# 以天安门附近坐标为中心，300km 内
GEOSEARCH cities FROMLONLAT 116.40 39.90 BYRADIUS 300 km ASC
# → beijing, tianjin
```



##### 5. 矩形范围：BYBOX

```java
# 以北京为中心，宽 1200km、高 1200km 的矩形
GEOSEARCH cities FROMMEMBER beijing BYBOX 1200 1200 km ASC
```



- `BYBOX 宽 高 单位`，宽是东西方向，高是南北方向

##### 6. COUNT + ANY 加速

```java
# 只要凑够 2 个就返回，不保证是最近的，但快
GEOSEARCH cities FROMMEMBER beijing BYRADIUS 1500 km ASC COUNT 2 ANY
```

#### 五、搜索并存储：GEOSEARCHSTORE（Redis 6.2+）

#####  基本用法：结果存成 ZSet

```java
# 语法：GEOSEARCHSTORE destination source <查询条件> [STOREDIST]
GEOSEARCHSTORE nearby cities FROMMEMBER beijing BYRADIUS 1500 km ASC
# 返回 (integer) 3 nearby是存入的Zset的key
```

查一下存了什么：

```java
TYPE nearby
# zset

ZRANGE nearby WITHSCORES
# 1) "beijing"    2) "4069091234567890"    ← score 是 geohash 编码
# 3) "tianjin"    4) "406988..."
# 5) "shanghai"   6) "406988..."
```

### 10.2 导入店铺数据

首先，店铺的mysql数据中存入了经纬度和店铺类型。教程业务中，把店铺按照不同的类型做了分类，比如：美食类、健身类等等..

![image-20261021163950547](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/image-20261021163950547.png)

#### 需求：

导入店铺数据时，需要按照店铺类型分块导入，但这些店铺在表结构上没有分表，只是用shop_type以作代替。

#### 解决：

1、按照店铺类型**分块获取**每种类型店铺各自的**经纬度及id**

2、使用GEO导入redis

涉及知识点：

- Stream流中终结方法`collect(Collectors.groupingBy())`可以将流里的元素按照某个类型进行分类，分类结果以map的形式输出，其中key为被指定分类的元素，value为具体的数据。解决了第一步
- ：GEOADD key 经度 纬度 member 

~~~java
@Test
void loadShopData() {
    // 1.查询店铺信息
    List<Shop> list = shopService.list();
    // 2.把店铺分组，按照typeId分组，typeId一致的放到一个集合
    Map<Long, List<Shop>> map = list.stream().collect(Collectors.groupingBy(Shop::getTypeId));
    // 3.分批完成写入Redis
    for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
        // 3.1.获取类型id
        Long typeId = entry.getKey();
        String key = SHOP_GEO_KEY + typeId;
        // 3.2.获取同类型的店铺的集合
        List<Shop> value = entry.getValue();
        List<RedisGeoCommands.GeoLocation<String>> locations = new ArrayList<>(value.size());
        // 3.3.写入redis GEOADD key 经度 纬度 member
        for (Shop shop : value) {
            // stringRedisTemplate.opsForGeo().add(key, new Point(shop.getX(), shop.getY()), shop.getId().toString());
            locations.add(new RedisGeoCommands.GeoLocation<>(
                    shop.getId().toString(),
                    new Point(shop.getX(), shop.getY())
            ));
        }
        stringRedisTemplate.opsForGeo().add(key, locations);
    }
}
~~~

### 10.3 查询附近店铺功能

#### 需求：

当点击商铺具体类别时，获取当下位置周围的商铺信息，距离从近到远。

![image-20261021213042766](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261021213042766.png)

![image-20261021213220961](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261021213220961.png)

#### 解决：

首先前端此时发送的请求为：携带参数typeId类别id、current当下页码（查询数量后端定义好默认为5个）、x、y经纬度（为测试方便写死了）

![image-20261021213331081](C:\Users\沨烨\AppData\Roaming\Typora\typora-user-images\image-20261021213331081.png)

**具体怎么干：**

1. 根据经纬度查询redis，获取当下页码对应的5个商铺id(member)和对应的距离，按照降序排列查出来

2. 查出（id,distance）的map集合；查出有序的id集合

3. 通过查出来的有序的商铺id列表在mysql中查出来对应有序的商铺信息，如图片、名称等信息传入对应的shop实体。通过map集合将对应的距离也写入shop实体中，返回shop实体。（这里的距离字段数据库是没有的，Shop类上进行了额外标注。）

4. ~~~java
   @TableField(exist = false)
   private Double distance;
   ~~~

**代码实现：**

~~~java
public Result queryShopByType(Integer typeId, Integer current, Double x, Double y) {
        //1.判断是否根据坐标查询。这里业务需求是：可以不传经纬度，如果不传经纬度就直接分页查询5个返回。
        if (x==null||y==null) {
            Page<Shop> page = query()
                    .eq("type_id", typeId)
                    .page(new Page<>(current, SystemConstants.DEFAULT_PAGE_SIZE));
            return Result.ok(page.getRecords());
        }
        //否，直接查
        //2.计算分页参数
        int from=(current-1)*SystemConstants.DEFAULT_PAGE_SIZE;
        int end=current*SystemConstants.DEFAULT_PAGE_SIZE;
        //3.查询redis、按照距离排序、分页。结果：shopId、distance GEOSEARCH key FROMLONLAT x y BYRADIUS 10 WITHDISTANCE;
        String key= SHOP_GEO_KEY+typeId;
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = stringRedisTemplate.opsForGeo().search(
                key,
                GeoReference.fromCoordinate(x, y),
                new Distance(5000),
                RedisGeoCommands.GeoSearchCommandArgs
                        .newGeoSearchArgs()
                        .includeDistance()//返回距离
                        .limit(end)//最多end个
        );
        //4.解析
        //健壮性判断
        if (results==null) {
            return Result.ok(Collections.emptyList());
        }
        List<GeoResult<RedisGeoCommands.GeoLocation<String>>> list = results.getContent();
        if(list.size()<=from){
            // 没有下一页了，结束
            return Result.ok(Collections.emptyList());
        }
        //4.1截取from-end 的数据
        Map<Long, @NonNull Distance> distanceMap = list.stream()
                .skip(from)
                .collect(Collectors.toMap(result -> Long.valueOf(result.getContent().getName())
                        , result -> result.getDistance()
                        ,(a,b)->a)
                        //,LinkedHashMap::new) // ← 第4个参数：按插入顺序的 Map
                        );

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
    }
~~~

**代码解析：**

**对应第一步：**查询距离当下坐标5km以内的最多**end**个的商铺id 、距离，以降序返回

注意GEO查询是没有分页查询的功能的，没办法做到查询对应页数中的5个商铺，所以这里直接将页码*5的数据全部查出来，等用到时再截取。

~~~java
//2.计算分页参数
        int from=(current-1)*SystemConstants.DEFAULT_PAGE_SIZE;
        int end=current*SystemConstants.DEFAULT_PAGE_SIZE;
        //3.查询redis、按照距离排序、分页。结果：shopId、distance GEOSEARCH key FROMLONLAT x y BYRADIUS 10 WITHDISTANCE;
        String key= SHOP_GEO_KEY+typeId;
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = stringRedisTemplate.opsForGeo().search(
                key,
                GeoReference.fromCoordinate(x, y),
                new Distance(5000),
                RedisGeoCommands.GeoSearchCommandArgs
                        .newGeoSearchArgs()
                        .includeDistance()//返回距离
                        .limit(end)//最多end个
        );
~~~

**对应第二步：**查出（id,distance）的map集合；查出有序的id集合

~~~java
 //4.解析
        //健壮性判断
        if (results==null) {
            return Result.ok(Collections.emptyList());
        }
        List<GeoResult<RedisGeoCommands.GeoLocation<String>>> list = results.getContent();
        if(list.size()<=from){
            // 没有下一页了，结束
            return Result.ok(Collections.emptyList());
        }
        //4.1截取from-end 的数据
    Map<Long, @NonNull Distance> distanceMap = list.stream()
                .skip(from)
                .collect(Collectors.toMap(result -> Long.valueOf(result.getContent().getName())
                        , result -> result.getDistance()
                        ,(a,b)->a
                        ,LinkedHashMap::new)// ← 第4个参数：按插入顺序的 Map
                        );

        //获取id集合
        /* List<Long> ids=new ArrayList<>(list.size());
        for (Long id : distanceMap.keySet()) {
            ids.add(id);
        }*/
        //用重构方法更简单
        ArrayList<Long> ids = new ArrayList<>(distanceMap.keySet());
~~~

解释一下这个stream流干了啥

~~~java
Map<Long, @NonNull Distance> distanceMap = list.stream()
        .skip(from) // 跳过前 from 条（内存分页）
        .collect(Collectors.toMap(
                // key：商铺 id 转 Long
                result -> Long.valueOf(result.getContent().getName()),
                // value：距离对象
                result -> result.getDistance(),
                // 合并函数：遇到重复 key 时保留先出现的，防崩溃
                (a, b) -> a,
                // 使用 LinkedHashMap 保证 Map 遍历顺序与 list 一致
                LinkedHashMap::new
        ))
~~~

**对应第三步**：通过查出来的有序的商铺id列表在mysql中查出来对应有序的商铺信息，如图片、名称等信息传入对应的shop实体。通过map集合将对应的距离也写入shop实体中，返回shop实体。

~~~java
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

## 11、用户签到

### Redis位映射类数据结构-BITMAP

教程原文档很详细，重点理解，bitmap如何输入，如何遍历。

**bitmap输入：**

~~~java
@Override
public Result sign() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.写入Redis SETBIT key offset 1
    stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
    return Result.ok();
}
~~~

![bitmap签到-写入](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/bitmap%E7%AD%BE%E5%88%B0-%E5%86%99%E5%85%A5.gif)

**bitmap遍历：**用1做**位与**运算，每次**运算后右移一位**。结果位1该位为1，结果为0该位为0。

~~~java
@Override
public Result signCount() {
    // 1.获取当前登录用户
    Long userId = UserHolder.getUser().getId();
    // 2.获取日期
    LocalDateTime now = LocalDateTime.now();
    // 3.拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern(":yyyyMM"));
    String key = USER_SIGN_KEY + userId + keySuffix;
    // 4.获取今天是本月的第几天
    int dayOfMonth = now.getDayOfMonth();
    // 5.获取本月截止今天为止的所有的签到记录，返回的是一个十进制的数字 BITFIELD sign:5:202203 GET u14 0
    List<Long> result = stringRedisTemplate.opsForValue().bitField(
            key,
            BitFieldSubCommands.create()
                    .get(BitFieldSubCommands.BitFieldType.unsigned(dayOfMonth)).valueAt(0)
    );
    if (result == null || result.isEmpty()) {
        // 没有任何签到结果
        return Result.ok(0);
    }
    Long num = result.get(0);
    if (num == null || num == 0) {
        return Result.ok(0);
    }
    // 6.循环遍历
    int count = 0;
    while (true) {
        // 6.1.让这个数字与1做与运算，得到数字的最后一个bit位  // 判断这个bit位是否为0
        if ((num & 1) == 0) {
            // 如果为0，说明未签到，结束
            break;
        }else {
            // 如果不为0，说明已签到，计数器+1
            count++;
        }
        // 把数字右移一位，抛弃最后一个bit位，继续下一个bit位
        num >>>= 1;
    }
    return Result.ok(count);
}
~~~

![bitmap签到-统计](https://natureshiyi-image.oss-cn-hangzhou.aliyuncs.com/img/bitmap%E7%AD%BE%E5%88%B0-%E7%BB%9F%E8%AE%A1.gif)

## 12、UV统计

### Redis数量统计类数据结构-HyperLogLog

**HyperLogLog（HLL）功能**：估算一个集合中**“不重复元素的总数（基数）”**。（解决：**海量数据统计（UV）**）

# 技术拓展

## 1.JWT验证登录

### 1.1 JWT介绍

#### 一句话：JWT 是「自带签名的身份证」

传统的 Session 是：服务端存一张表（`sessionId → 用户信息`），客户端只拿一个毫无意义的 ID。

JWT 反过来：**用户信息直接写在令牌里，服务端靠一个密钥验证"这令牌是不是我签发的"**，不需要存任何东西。

------

#### 结构：三段，用 `.` 隔开

```java
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiIxMDAxIiwiZXhwIjoxNzMwMDAwMDAwfQ . 4kQ2s...
        ↑ Header（头）                        ↑ Payload（载荷）                    ↑ Signature（签名）
```

#### 第一段 Header：用什么算法签的

Base64Url 解码后：

```json
{"alg":"HS256","typ":"JWT"}
```

#### 第二段 Payload：装了什么数据

```json
{
  "sub": "1001",        ← subject，一般放 userId
  "role": "user",
  "exp": 1730000000,    ← 过期时间戳（秒）
  "iat": 1729998200     ← 签发时间
}
```

**你可以往里塞任意字段**，比如 `{"userId":1001,"nickname":"沨烨"}`。

#### 第三段 Signature：防伪的核心

```java
signature = HMACSHA256(
    base64(header) + "." + base64(payload),
    服务端保管的密钥
)
```

**关键点：签名需要密钥，而密钥只在服务端。** 所以：

- 攻击者能改 payload（它是明文），但**改完签名就对不上了**
- 攻击者能伪造一个新 payload，但没有密钥，**签不出合法的 signature**

### 1.2 JWT实战使用

主要就两个步骤

- 生成`JWTToken`返回前端(传入Header、Payload、Signature用于生成)
- 解析前端发送来的`JWTToken`进行验证

**生成时一般用于用户进行登录后返回给前端，解析时一般用于是拦截器端获取JWTToken进行登录状态验证、身份验证等**



pom.xml

~~~xml
 <!--JWT        -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.12.6</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
~~~

把jwt生成token和解析token做成工具类以便调用

//注意密钥用`Keys.hmacShaKeyFor`生成时定义的字符数量要超过32位

~~~java
public class JWTUtils {

    /**
     * 密钥。
     * HS256 要求至少 32 字节（256 bit），这里字符串是 40 个字符，够用。
     * Keys.hmacShaKeyFor() 会把 byte[] 包装成 HMAC 用的 SecretKey。
     */
    private static final SecretKey SECRET_KEY =
            Keys.hmacShaKeyFor("secretkeysecretkeysecretkey1234567890".getBytes());

    /**
     * 生成 token。
     * 返回的字符串长这样：header.payload.signature（三段，用 . 隔开）
     */
    public static String createToken(UserDTO user) {

        JwtBuilder builder = Jwts.builder();   // ① 拿到构建器，此时还是空的

        String JWTtoken = builder
                // ---------- ② Header（头）：说明用什么算法 ----------
                .setHeaderParam("type", "JWT")          // 声明这是个 JWT
                .setHeaderParam("alg", "HS256")         // 声明签名算法是 HS256

                // ---------- ③ Payload（载荷）：装业务数据 ----------
                .setSubject("user")                     // sub 字段，这里填的是字符串 "user"
                .claim("id", user.getId().longValue())  // 自定义字段 id
                .claim("nickname", user.getNickName())  // 自定义字段昵称
                .claim("icon", user.getIcon())          // 自定义字段头像地址
                .setIssuedAt(new Date())                // iat：签发时间，现在就发
                .setExpiration(new Date(
                        System.currentTimeMillis() + 8 * 60 * 60 * 1000))
                                                        // exp：8 小时后过期
                                                        // 8*60*60*1000 = 28,800,000 毫秒

                // ---------- ④ Signature（签名）：防伪 ----------
                .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
                // 用密钥对 "base64(header).base64(payload)" 算一遍 HMAC-SHA256
                // 结果贴在最后一段

                .compact();                             // ⑤ 拼成最终字符串

        return JWTtoken;
    }

    /**
     * 解析 token。
     * 做了三件事：验签 → 校验过期 → 把 payload 反序列化成 Claims
     * 任何一步失败都会抛 JwtException 的子类，调用方必须捕获。
     */
    public static Claims parseToken(String JWTToken) {
        return Jwts.parser()
                .verifyWith(SECRET_KEY)          // 用什么密钥验签
                .build()                          // 构建出解析器
                .parseSignedClaims(JWTToken)      // 拆三段 + 验签 + 查过期
                .getPayload();                    // 取出 payload
    }
}
~~~



**使用：**

#### 登录后生成token返回

~~~java
public Result loginWithJWT(LoginFormDTO loginForm, HttpSession session) {
        //1.验证手机格式
        String phone = loginForm.getPhone();
        if (RegexUtils.isPhoneInvalid(phone)) {
            //2.不符合返回错误信息
            return Result.fail("手机号格式错误！");
        }

        //3.从redis中获取验证码并校验
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
        String code = loginForm.getCode();
        if (cacheCode == null || !cacheCode.toString().equals(code)) {
            //3.1不一致报错
            return Result.fail("验证码错误");
        }
        //3.2一致，根据手机号查用户
        User user = query().eq("phone", phone).one();
        //4.判断用户是否存在
        if (user == null) {
            //4.1不存在注册新用户，数据保存到数据库
            user = createUserWithPhone(phone);
        }
        //5.格式转换，User->UserDTO
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        //6.通过JWT生成token
        String token = JWTUtils.createToken(userDTO);
        return Result.ok(token) ;
    }

~~~

#### 定义拦截器拦截解析

~~~java
package com.hmdp.utils;

import com.hmdp.dto.UserDTO;
import io.jsonwebtoken.Claims;
import org.springframework.lang.Nullable;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class JWTInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null){
            return false; // Token is missing
        }
        //解析token
        Claims claims = JWTUtils.parseToken(token);
        if(claims == null){
            return false; // Token is invalid
        }
        //保存用户信息到ThreadLocal
        UserDTO userDTO = new UserDTO();
        userDTO.setId(claims.get("id", Long.class));
        userDTO.setNickName(claims.get("nickname", String.class));
        userDTO.setIcon(claims.get("icon", String.class));
        UserHolder.saveUser(userDTO);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        UserHolder.removeUser();
    }
}

~~~

