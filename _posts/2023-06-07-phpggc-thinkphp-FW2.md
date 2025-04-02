---
title: phpggc thinkphp FW2
date: 2023-06-07 22:21:49
categories:
- PHP
tags:
- Deserialization
- thinkphp
toc: true
image:
  path: /assets/images/php.png
notshow: true
---

> - [php-反序列化格式基础](/posts/php-反序列化格式基础.html)
- [php-反序列化绕过](/posts/php-反序列化绕过.html)
- [php-反序列化字符逃逸](/posts/php-反序列化字符逃逸.html)
- [php-phar-反序列化](/posts/php-phar-反序列化利用.html)
- [php-反序列化原生类利用](/posts/php-反序列化原生类利用.html)
- [phpggc-利用链分析](/posts/phpggc-thinkphp-利用链分析.html)
  - thinkphp
    - [phpggc-thinkphp-RCE1](/posts/phpggc-thinkphp-RCE1.html)
    - [phpggc-thinkphp-RCE2](/posts/phpggc-thinkphp-RCE2.html)
    - [phpggc-thinkphp-RCE3](/posts/phpggc-thinkphp-RCE3.html)
    - [phpggc-thinkphp-RCE4](/posts/phpggc-thinkphp-RCE4.html)
    - [phpggc-thinkphp-FW1](/posts/phpggc-thinkphp-FW1.html)
    - [phpggc-thinkphp-FW2](/posts/phpggc-thinkphp-FW2.html)


## FW2
影响范围：
- thinkphp 5.0.0-5.0.3

### 使用
FW2 使用时需要提供两个参数，第一个参数用于指定写入的路径，第二个参数用于指定本地读取的文件。-b 参数用于将输出转为 base64。

```bash
phpggc ThinkPHP/FW2 /tmp/ /mnt/share/Tools/web/PHP/unserialize/phpggc/test/test.txt -b
```

### 分析
整个调用链如下所示：

```php
think\Process::__destruct()                                
    | $this->close()                                       
    | $this->processPipes->close()                         
    think\model\Relation::__call()  <---- [diff]                       
        | $this->query->where()                                     
        think\console\Output::__call()                     
            | call_user_func_array([$this, 'block'], $args)
            | $this->block()                               
            | $this->handle->write()                       
            think\session\driver\Memcache::write()         
                | $this->handler->set()                    
                think\cache\driver\Memcached::set()<-+     
                    | $this->handler->set()          |     
                    think\cache\driver\File::set()   |     
                        | $this->setTagItem()        |     
                        | $this->set()               |     
                        file_put_contents() * <----  |     
                    | $this->setTagItem()            |     
                    | $this->set()-------------------+       
```

对比 FW2 与 FW1 的 gadget 可以发现两者大部分是一样的，但 FW1 中利用的是： `think\model\relation\HasMany`
```php
    think\model\relation\HasMany::__call()                         
        | $this->baseQuery()                               
        | $this->query->where()        
```
而 FW2 中直接利用了`think\model\Relation` 类。

区别的原因在于 `think\model\Relation` 类 __call 中的内容发生了改变, 在 5.0.3 中 Relation 类可以直接调用 `$this->query->where()`
```php
public function __call($method, $args)
{
    if ($this->query) {
        switch ($this->type) {
            case self::HAS_MANY:
                if (isset($this->where)) {
                    $this->query->where($this->where);
```
而在 5.0.23 中，Relation 的操作用几个子类来完成,`$this->baseQuery` 会调用子类的 baseQuery 方法，FW1 中利用的 `think\model\relation\HasMany` 就是 Relation 的子类。
```php
public function __call($method, $args)
{
    if ($this->query) {
        // 执行基础查询
        $this->baseQuery();
```
