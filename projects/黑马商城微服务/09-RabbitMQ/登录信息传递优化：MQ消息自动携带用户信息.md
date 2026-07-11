---
date: 2026-07-11
tags: [RabbitMQ, 黑马商城, UserContext, 自动装配]
---

# 登录信息传递优化：MQ 消息自动携带用户信息

## 一、写在前面

MQ 异步消费时，`UserContext`（ThreadLocal）拿不到用户信息——因为消息在另一个线程处理。之前的做法是在消息体里手动传 `userId`，消费者再手动取。

问题：每个消息体都要加 `userId` 字段，每个消费者都要手动 `setUser`，跟 OpenFeign 的体验不统一。

目标：**业务代码无感知**，生产者自动写 Header，消费者自动读 Header 设置 `UserContext`。

## 二、实现原理

```
生产者（发送前）                    消费者（执行前）
UserContext.getUser()    →    Header: user-info=1    →    UserContext.setUser(1)
       ↓                                                          ↓
   业务代码无感知                                            业务代码无感知
   直接 rabbitTemplate.send()                            直接 UserContext.getUser()
```

跟 OpenFeign 传用户信息一个思路：
- OpenFeign：`RequestInterceptor` 写 Header → `UserInfoInterceptor` 读 Header
- MQ：`BeanPostProcessor` 写 Header → `MethodInterceptor` 读 Header

## 三、核心代码：MqConfig

放在 `hm-common` 模块，所有微服务自动生效。

### 3.1 生产者：自动写 Header

```java
@Bean
public static BeanPostProcessor rabbitTemplatePostProcessor() {
    return new BeanPostProcessor() {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            if (bean instanceof RabbitTemplate) {
                RabbitTemplate rabbitTemplate = (RabbitTemplate) bean;
                rabbitTemplate.addBeforePublishPostProcessors(message -> {
                    Long userId = UserContext.getUser();
                    if (userId != null) {
                        message.getMessageProperties().setHeader("user-info", userId);
                    }
                    return message;
                });
            }
            return bean;
        }
    };
}
```

`addBeforePublishPostProcessors`：在消息发送前自动执行，把 `UserContext.getUser()` 写入消息 Header。

### 3.2 消费者：自动读 Header

```java
@Bean
public static BeanPostProcessor rabbitListenerContainerFactoryPostProcessor() {
    return new BeanPostProcessor() {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            if (bean instanceof SimpleRabbitListenerContainerFactory) {
                SimpleRabbitListenerContainerFactory factory = (SimpleRabbitListenerContainerFactory) bean;
                Advice advice = new UserContextMethodInterceptor();
                // 追加到已有 adviceChain，不覆盖
                Advice[] existing = factory.getAdviceChain();
                if (existing != null) {
                    Advice[] newChain = new Advice[existing.length + 1];
                    System.arraycopy(existing, 0, newChain, 0, existing.length);
                    newChain[existing.length] = advice;
                    factory.setAdviceChain(newChain);
                } else {
                    factory.setAdviceChain(advice);
                }
            }
            return bean;
        }
    };
}
```

### 3.3 MethodInterceptor 实现

```java
static class UserContextMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 从方法参数中获取 Message，提取 user-info
        for (Object arg : invocation.getArguments()) {
            if (arg instanceof Message) {
                Message message = (Message) arg;
                Long userId = message.getMessageProperties().getHeader("user-info");
                if (userId != null) {
                    UserContext.setUser(userId);
                }
                break;
            }
        }
        try {
            return invocation.proceed();
        } finally {
            UserContext.removeUser(); // 防止内存泄漏
        }
    }
}
```

`MethodInterceptor` 继承自 `Advice`，可以设置到 `SimpleRabbitListenerContainerFactory.setAdviceChain()`。

## 四、自动装配

`spring.factories` 注册：
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.hmall.common.config.MyBatisConfig,\
  com.hmall.common.config.MvcConfig,\
  com.hmall.common.config.MqConfig
```

`@ConditionalOnClass(RabbitTemplate.class)`：只有引入了 AMQP 依赖的服务才生效，没用 MQ 的服务不受影响。

## 五、改造前后对比

### 消息体（OrderMessage）

```java
// 改造前：手动传 userId
@Data
public class OrderMessage implements Serializable {
    private List<Long> itemIds;
    private Long userId;  // 手动传
}

// 改造后：userId 走 Header，消息体只关心业务数据
@Data
public class OrderMessage implements Serializable {
    private List<Long> itemIds;
}
```

### 生产者（OrderServiceImpl）

```java
// 改造前
orderMessage.setUserId(UserContext.getUser());
rabbitTemplate.convertAndSend("trade.topic", "order.create", orderMessage);

// 改造后（无感知）
rabbitTemplate.convertAndSend("trade.topic", "order.create", orderMessage);
```

### 消费者（ClearCartListener）

```java
// 改造前：从消息体取 userId
.eq(Cart::getUserId, message.getUserId())

// 改造后：直接用 UserContext（无感知）
.eq(Cart::getUserId, UserContext.getUser())
```

## 六、踩坑避坑指南

### 坑 1：RabbitTemplateCustomizer 找不到

- 原因：`RabbitTemplateCustomizer` 在 `spring-boot-autoconfigure` 中，hm-common 没有直接依赖
- 解决：改用 `BeanPostProcessor` 拦截 `RabbitTemplate` Bean

### 坑 2：Advice 类型不兼容

- 原因：`AbstractPointcutAdvisor` 是 `Advisor`，不是 `Advice`
- 解决：直接用 `MethodInterceptor`（它继承自 `Interceptor` 继承自 `Advice`）

### 坑 3：adviceChain 覆盖问题

- 问题：直接 `setAdviceChain()` 会覆盖已有配置
- 解决：先 `getAdviceChain()` 取出已有数组，追加新元素

## 七、全文总结

- 核心思路：**Header 传用户，跟 OpenFeign 一个套路**
- 生产者：`BeanPostProcessor` + `addBeforePublishPostProcessors` 自动写 Header
- 消费者：`BeanPostProcessor` + `setAdviceChain` + `MethodInterceptor` 自动读 Header
- 业务代码完全无感知，统一用 `UserContext.getUser()`
- 放在 `hm-common`，`@ConditionalOnClass` 控制生效范围

## 八、关联笔记

- [[业务改造：同步调用改MQ异步通知]]
- [[SpringAMQP入门与整合]]

## 九、代码变更

- 新增：`hm-common/MqConfig.java` — MQ 自动装配（生产者写 Header + 消费者读 Header）
- 修改：`hm-common/spring.factories` — 注册 MqConfig
- 修改：`hm-api/OrderMessage.java` — 移除 userId 字段
- 修改：`trade-service/OrderServiceImpl.java` — 移除 setUserId
- 修改：`cart-service/ClearCartListener.java` — 改用 UserContext.getUser()
