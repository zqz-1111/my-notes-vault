---
date: 2026-07-14
tags: [snippets, OpenFeign, 微服务]
---

# FeignClient 查询单个对象

## 用途

其他微服务通过 Feign 远程调用 item-service 查询单个商品。

## 代码

**ItemClient.java**（hm-api 模块）：

```java
@FeignClient(value = "item-service", fallbackFactory = ItemClientFallback.class)
public interface ItemClient {

    @GetMapping("/items/{id}")
    ItemDTO queryItemById(@PathVariable("id") Long id);

    @GetMapping("/items")
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids);

    @PutMapping("/items/stock/deduct")
    void deductStock(@RequestBody List<OrderDetailDTO> items);

    @PutMapping("/items/stock/restore")
    void restoreStock(@RequestBody List<OrderDetailDTO> items);
}
```

**ItemClientFallback.java**（降级逻辑）：

```java
@Override
public ItemDTO queryItemById(Long id) {
    log.error("远程调用ItemClient#queryItemById方法出现异常，参数：{}", id, cause);
    return null;
}
```

## 注意事项

- `@PathVariable` 要加，不然 URL 拼接不上
- 降级返回 `null`（查询单个商品失败返回 null 比抛异常更安全）
- `@GetMapping` 的路径要跟 item-service 的 Controller 一致
