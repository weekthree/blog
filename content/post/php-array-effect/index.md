
+++
author = "week three"
title = "Array 函数效率问题"
date = "2022-06-10"
description = "不仅仅是慢速 SQL 会导致我们的程序变慢，不恰当的内置函数的使用也是需要重视的一个点。要在不同的场景选择适用的内置函数。"
categories = [
    "php"
]
tags = [
    "php",
    "array"
]
draft = false
image = "1.jpg"
+++

## 起因
PHP Array 函数是 PHP 核心的组成部分，我们在编写 PHP 代码的时候，这些内置的 Array 函数给我们提供了极大的便利。但是有一次我发现在测试环境执行无异常的代码，在线上执行却超时了。
开始以为是代码里面有 SQL 慢查询导致的，但是排查相关的数据库语句并没有发现什么问题。最后定位到一段遍历语句当中，里面用到了三个数组函数，分别是数组合并函数`array_merge()`，数组查找函数`array_search()`和`in_array()`。
我们先看下面这段垃圾代码。
```php

foreach ($poolStr as $poolIds) {
    $poolId = explode(',', $poolIds);
    $batchPool = array_merge($batchPool, $poolId);
}
...
foreach ($batchPool as $bp) {
    $postId = array_search($bp,$turnPool);
    if ($postId && in_array($postId,$onlinePost)) {
        //do somgthing
    }
}
```
通过分析，意识到虽然这里并没有相关的数据库查询操作，但是遍历可能涉及到大量的循环，相关内置函数的效率问题会明显放大。
下面用例子证明一下。（使用相同的`range()`函数创建一样长度的数组，这次不考虑`range()`函数的时间消耗。)
## Array 函数
### array_merge()
我们循环10000次，每次生成10个元素的数组，然后使用`array_merge()`函数合并成一个大数组。经过测试发现，10000次循环合并就要花费4秒多。
```php
<?php
/**
 * 获取毫秒
 */
if (!function_exists('getMsectime')) {
    function getMsectime()
    {
        list($msec, $sec) = explode(' ', microtime());
        return (float)sprintf('%.0f', (floatval($msec) + floatval($sec)) * 1000);
    }
}


$time_s = getMsectime();
$arr = [];
for ($i=0; $i<=10000; $i++){
    $newArr = range(0,10);
    $arr = array_merge($arr,$newArr);
}
$time_e = getMsectime();
$time_u = $time_e - $time_s;
echo "时间开销:{$time_u}ms     ";
echo "元素个数".count($arr);

//输出结果
//时间开销:4368ms     元素个数110011
```
因为我们不关注键名，可以直接使用`array_push`替代，得到相同的数据，时间只需要5毫秒。`array_merge()`函数与其相比，性能相差了千倍。
```php
$time_s = getMsectime();
$arr = [];
for ($i=0; $i<=10000; $i++){
    $newArr = range(0,10);
    array_push($arr,...$newArr);
}
$time_e = getMsectime();
$time_u = $time_e - $time_s;
echo "时间开销:{$time_u}ms     ";
echo "元素个数".count($arr);

//输出结果
//时间开销:5ms     元素个数110011
```
### array_search()
使用`range()`函数生成1-9999的数字元素的数组。使用`array_search()`执行50万次查找操作，花费了将近7秒的时间。
```php
$time_s = getMsectime();
$arr = [];
$newArr = range(1,9999);
$nums = 0;
for ($i=0; $i<=500000; $i++){
    if(array_search($i,$newArr) !== false){
        $nums++;
    }
}
$time_e = getMsectime();
$time_u = $time_e - $time_s;
echo "时间开销:{$time_u}ms     ";
echo "存在元素个数{$nums}个";
//输出结果
//时间开销:6696ms     存在元素个数9999个
```
### in_array()
与`array_search()`半斤八两的`in_array()`
```php
$time_s = getMsectime();
$arr = [];
$newArr = range(1,9999);
$nums = 0;
for ($i=0; $i<=500000; $i++){
    if(in_array($i,$newArr) !== false){
        $nums++;
    }
}
$time_e = getMsectime();
$time_u = $time_e - $time_s;
echo "时间开销:{$time_u}ms     ";
echo "存在元素个数{$nums}个";
//输出结果
//时间开销:5932ms     存在元素个数9999个
```

我们进行如下优化，使用`array_flip()`先将整个数组进行反转。然后使用`array_key_exists()`在反转后的数组中查找需要搜索的key，大大提升搜索的效率。
当然，这里也可以使用`isset()`，它使用了与`array_key_exists()`相同的操作，而且，由于`isset()`属于语言结构，在重复使用相同 key 的情况下会利用缓存提升效率。经过测试，`isset()`确实比`array_key_exists()`快那么一丢丢。
下面是时间复杂度相关的资料，`in_array()` 和`array_search()`都是`O(n)`，`array_key_exists()`和`isset()`为`O(1)`。
> 1. array_key_exists O(n) but really close to O(1) - this is because of linear polling in collisions, but because the chance of collisions is very small, the coefficient is also very small. I find you treat hash lookups as O(1) to give a more realistic big-O. For example the different between N=1000 and N=100000 is only about 50% slow down.
> 1. isset( $array[$index] ) O(n) but really close to O(1) - it uses the same lookup as array_key_exists. Since it's language construct, will cache the lookup if the key is hardcoded, resulting in speed up in cases where the same key is used repeatedly.
> 1. in_array O(n) - this is because it does a linear search though the array until it finds the value.
> 1. array_search O(n) - it uses the same core function as in_array but returns value.

```php
$time_s = getMsectime();
$arr = [];
$newArr = range(1,9999);
$nums = 0;
$newArr = array_flip($newArr);
for ($i=0; $i<=500000; $i++){
    if(array_key_exists($i,$newArr)){
        $nums++;
    }
    /* 使用isset也可以
     if(isset($newArr[$i]) !== false){
        $nums++;
    }
    */
}
$time_e = getMsectime();
$time_u = $time_e - $time_s;
echo "时间开销:{$time_u}ms     ";
echo "存在元素个数{$nums}个";
//输出结果
//array_key_exists
//时间开销:25ms     存在元素个数9999个

//isset
//时间开销:18ms     存在元素个数9999个
```
先使用`array_flip()`先将整个数组进行反转，再使用`isset()`或者`array_key_exists()`替代数组搜索函数，可以大幅提高执行效率。
## 总结
过上面三个例子，我们发现不仅仅是慢速 SQL 会导致我们的程序变慢，不恰当的内置函数的使用也是需要重视的一个点。那么如何在享受内置函数便利的同时，又规避性能陷阱呢？
这就需要我们对日常高频使用的相关函数的时间复杂度和执行方式都要有个大致的了解，这样才能在不同的场景中找到最适合的函数，也能在出现问题后快速定位，寻找别的实现方式替代。
## 参考资料
[List of Big-O for PHP functions](https://stackoverflow.com/questions/2473989/list-of-big-o-for-php-functions)  
[PHP 手册- Manual](https://www.php.net/manual/zh/function.in-array.php)
