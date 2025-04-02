---
title: phpggc thinkphp RCE2
date: 2023-06-07 22:21:19
categories:
- PHP
tags:
- Deserialization
- thinkphp
toc: true
image:
  path: /assets/images/php.png
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


## RCE2
影响范围：
- thinkphp 5.0.24
其他 5.0.x 的版本也可以适用。


### 使用

```bash
phpggc ThinkPHP/RCE2 system id  -b -u
```

### 分析
分析完 RCE5 才看的 RCE2, 详细的分析可见 RCE5. 两者后面部分基本一致，利用 Request 类中的 call_user_func 来执行 system。在 RCE1 的分析中，5.1.x 版本可以通过闭包函数来执行 system 函数，但 5.0.x 中并没有 `think\model\concern\Conversion` 和`think\model\concern\Attribute` 这两个 trait。

RCE2 利用链的起始部分于 RCE1 相似。区别在与 RCE1 调用 getAttr 时就可以走到 sink 点，而 RCE2 用来触发`think\console\Output::__call()`方法。

整个调用链如下所示：
```php
think\process\pipes\Windows::__destruct()                                
    | $this->removeFiles();                                    
    | file_exists()                     
    think\model\concern\Conversion::__toString()
        | $this->toJson()
        | $this->toArray()
        | $this->getAttr()              <---- same as RCE1 above     
        think\console\Output::__call()   <---- same as RCE5 below                  
            | call_user_func_array([$this, 'block'], $args)
            | $this->block()      
            | $this->handle->write()                       
            think\session\driver\Memcache::write()         
                | $this->handler->set()                    
                think\cache\driver\Memcached::set() 
                    | $this->has() 
                    | $this->handler->get()      
                    think\Request::get()      
                        | $this->input()        
                        | $this->filterValue()            
                        call_user_func() * <----        
```