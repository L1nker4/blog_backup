---
title:  理解PHP变量覆盖漏洞
date: 2018-11-01 11:32:14
categories:
- Web安全
- CTF
tags:
- PHP代码审计
- 安全
- CTF
---

<!--more-->

## 0x01 简介

变量覆盖漏洞是指攻击者使用自定义的变量去覆盖源代码中的变量，从而改变代码逻辑，实现攻击目的的一种漏洞。

通常造成变量覆盖的往往是以下几种情况：
- register_globals=On （PHP5.3废弃，PHP5.4中已经正式移除此功能）
- 可变变量
- extract()
- parse_str()
- import_request_variables()
下面正式介绍变量覆盖的具体利用方法与原理。

## 0x02 正文

### $可变变量
看下面一段代码：
```PHP
<?php
foreach (array('_COOKIE','_POST','_GET') as $_request)  
{
    foreach ($$_request as $_key=>$_value)  
    {
        $$_key=  $_value;
    }
}
$id = isset($id) ? $id : 2;
if($id == 1) {
    echo "flag{xx}";
    die();
}
?>
```
使用foreach遍历COOKIE，POST，GET数组，然后将数组键名作为变量名，键值作为变量值。上述代码传入id=1即可得到flag。


### extract()
首先看看手册中对该函数的定义
![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/php-cover/cover1.jpg)

下面从实例讲起:
```PHP
<?php
$flag='flag{xxxxx}'; 
extract($_GET);
 if(isset($a)) { 
    $content=trim(file_get_contents($flag));
    if($a==$content)
    { 
        echo $flag; 
    }
   else
   { 
    echo'Oh.no';
   } 
   }else {
    echo "input a value";
   }
?>
```
首先extract函数将所有GET方法得到的数组键名与键值转化为内部变量与值，接可以看到content变量使用file_get_contents()读入一个文件，由于trim()函数处理后，content变量变为一个空字符串，代码逻辑只需要**a == content**即可。

传入 **?a=** 即可得到flag。

### parse_str()
![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/cover3.jpg)

```PHP
<?php
parse_str("a=1");
echo $a."<br/>";      //$a=1
parse_str("b=1&c=2",$myArray);
print_r($myArray);   //Array ( [b] => 1 [c] => 2 ) 
?>
```
这个简单的实例中可以看出parse_str可以将字符串解析为变量与值。

### import_request_variables()

![](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/import.jpg)

```PHP
<?php
// 此处将导入 GET 和 POST 变量
// 使用"rvar_"作为前缀
$rvar_foo = 1;
import_request_variables("gP", "rvar_");
echo $rvar_foo;
?> 
```
通过import_request_variables函数作用后，可以对原变量值进行覆盖。

## 0x03 总结
博客文章以后会在公众号上同步发布，欢迎师傅们关注。微信号：coding_lin