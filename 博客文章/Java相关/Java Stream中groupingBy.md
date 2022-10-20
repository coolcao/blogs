---
title: 深入理解Java Stream中groupingBy分组统计
date: 2022-10-19 18:45:36
tags: [java, stream]
categories:
  - 技术博客
  - 原创
---

Java的Stream有一个非常重要且有用的操作符 `groupingBy`，可以对数据进行分组统计。

我们可以根据业务需要，对分组的键以及分组规则进行自定义，以实现更复杂的分组统计计算。

<!-- more -->

## 简单分组统计
一个非常简单的例子，假设我们有一个名字的列表，我们想统计一下每个名字有多少人重名：

```java
List<String> names = Arrays.asList(
"Tom", "Jack", "Lily", "Alex", 
"Jame", "Jack", "Lucy", "Alex", 
"Tom", "Lily", "Lucy", "Jame", "Lucy");

Map<String, Long> namesCount = names
    .stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
System.out.println(namesCount);

// 结果
// {Alex=2, Tom=2, Lucy=3, Jame=2, Jack=2, Lily=2}
```

`groupingBy()`操作符接收两个参数，第一个是指定分组的 `key`，第二个参数是实现聚合计算的下游流。

## 稍复杂的结构
如果我们的数据结构稍微复杂点，比如是一个类：

```java
List<ProductSold> datas = new ArrayList<>();
datas.add(new ProductSold("电脑", "数码产品",2999, 156));
datas.add(new ProductSold("键盘", "数码产品",199, 2390));
datas.add(new ProductSold("鼠标", "数码产品",159, 689));
datas.add(new ProductSold("移动硬盘", "数码产品",799, 231));
datas.add(new ProductSold("空调", "家电",1589, 463));
datas.add(new ProductSold("净水器", "家电",588, 309));
datas.add(new ProductSold("冰箱", "家电",2588, 778));
datas.add(new ProductSold("自行车", "交通",1549, 499));
datas.add(new ProductSold("电动车", "交通",2549, 482));

System.out.println(JSON.toJSONString(datas));

// 按分类category进行分组统计
Map<String, Long> categoryResult = datas.stream()
    .collect(Collectors.groupingBy(ProductSold::getCategory, Collectors.counting()));
System.out.println(categoryResult);

// 结果
// {数码产品=4, 家电=3, 交通=2}
```

## 自定义聚合规则
还是上面的数据，如果我们要按 `category` 进行分组，统计每个种类商品的销量，销售额，占比等，则需要重写聚合规则，即第二个参数：

```java
// 按category进行分组统计，销量，销售额，占比等
Map<String, CategoryResult> categoryResultMap = datas.stream()
    .collect(Collectors.groupingBy(
        ProductSold::getCategory,
        Collectors.collectingAndThen(Collectors.toList(), list -> {
            long count = list.stream().count();
            double soldAmount = list.stream().map(item -> item.getPrice()*item.getSoldCount()).reduce(0d, (sum, item) -> sum+item);
            String precent = BigDecimal.valueOf(count).divide(BigDecimal.valueOf(datas.size()), 4, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100)).toString() + "%";

            return new CategoryResult(count, soldAmount, precent);
        })
));
System.out.println(categoryResultMap);

// 结果
// {数码产品=CategoryResult{soldCount=4, soldAmount=1237574.0, precent='44.4400%'}, 家电=CategoryResult{soldCount=3, soldAmount=2930863.0, precent='33.3300%'}, 交通=CategoryResult{soldCount=2, soldAmount=2001569.0, precent='22.2200%'}}

```

## 自定义分组的key
还是上面的数据，假如我们不按商品种类分，而是按商品售价，统计一下售价区间：

```java
// 按价格区间分组，自定义分组key
List<String> priceRanges = Arrays.asList("0-500", "501-1000", "1001-2000", "2001-3000", ">3000");

Map<String, ProductSoldResult> resultMap = datas.stream().collect(Collectors.groupingBy(item -> {
    double price = item.getPrice();
    String key = priceRanges.get(0);

    if (price>500 && price <= 1000) key = priceRanges.get(1);
    if (price>1001 && price <= 2000) key = priceRanges.get(2);
    if (price>2001 && price <= 3000) key = priceRanges.get(3);
    if (price > 3000) key = priceRanges.get(4);
    return key;

}, Collectors.collectingAndThen(Collectors.toList(), list -> {
    long count = list.stream().count();
    double soldAmount = list.stream().map(item -> item.getPrice()*item.getSoldCount()).reduce(0d, (sum, item) -> sum+item);

    ProductSoldResult result = new ProductSoldResult();
    result.setSoldAmount(soldAmount);
    result.setSoldCount((int)count);
    result.setPrecent(BigDecimal.valueOf(count).divide(BigDecimal.valueOf(datas.size()), 4, RoundingMode.HALF_UP).multiply(BigDecimal.valueOf(100)).toString() + "%");
    result.setProductSolds(list);

    return result;
})));


for (Map.Entry<String, ProductSoldResult> entry: resultMap.entrySet()) {
    System.out.println(entry.getKey() + ":" + entry.getValue());
}

// 结果
// 1001-2000:ProductSoldResult{soldCount=2, soldAmount=1508658.0, precent='22.2200%', productSolds=[ProductSold{name='空调', category='家电', price=1589.0, soldCount=463}, ProductSold{name='自行车', category='交通', price=1549.0, soldCount=499}]}
// 501-1000:ProductSoldResult{soldCount=2, soldAmount=366261.0, precent='22.2200%', productSolds=[ProductSold{name='移动硬盘', category='数码产品', price=799.0, soldCount=231}, ProductSold{name='净水器', category='家电', price=588.0, soldCount=309}]}
// 0-500:ProductSoldResult{soldCount=2, soldAmount=585161.0, precent='22.2200%', productSolds=[ProductSold{name='键盘', category='数码产品', price=199.0, soldCount=2390}, ProductSold{name='鼠标', category='数码产品', price=159.0, soldCount=689}]}
// 2001-3000:ProductSoldResult{soldCount=3, soldAmount=3709926.0, precent='33.3300%', productSolds=[ProductSold{name='电脑', category='数码产品', price=2999.0, soldCount=156}, ProductSold{name='冰箱', category='家电', price=2588.0, soldCount=778}, ProductSold{name='电动车', category='交通', price=2549.0, soldCount=482}]}
```





