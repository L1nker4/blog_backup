---
title: PHP弱类型产生的安全问题
date: 2018-11-09 11:00:15
categories:
- Web安全
- CTF
tags:
- PHP代码审计
- 安全
- CTF
---

<!--more-->

# 简介

PHP作为一种弱类型编程语言，在定义变量时无需像C++等强类型时定义数据类型。
```PHP
<?php
$a = 1;
$a = "hello";
$a = [];
?>
```
上述代码可以正常运行，在PHP中可以随时将变量改成其他数据类型。下面再来看一个例子：
```PHP
<?php
$a = '1';   //a现在是字符串'1'
$a *= 2;    //a现在是整数2
?>
```
下面就从具体的实例中总结一下弱类型产生的安全问题。
# 比较运算符

## 简介
PHP有这些比较运算符
```PHP
<?php
$a == $b;   //相等
$a === $b;  //全等
$a != $b;   //不等
$a <> $b;   //不等
$a !== $b;   //不全等
$a < $b;   //小于
$a > $b;   //大于
$a <= $b;   //小于等于
$a >= $b;   //大于等于
$a <==> $b;   //太空船运算符：当$a小于、等于、大于$b时分别返回一个小于、等于、大于0的integer 值。 PHP7开始提供.  
$a ?? $b ?? $c;   //NULL 合并操作符 从左往右第一个存在且不为 NULL 的操作数。如果都没有定义且不为 NULL，则返回 NULL。PHP7开始提供。 
?>
```
存在安全问题的一般都是运算符在类型转换时产生的。给出PHP手册中比较的表格。
> http://php.net/manual/zh/types.comparisons.php

下面看一段从PHP手册中摘的一段：
>如果该字符串没有包含 '.'，'e' 或 'E' 并且其数字值在整型的范围之内（由 PHP_INT_MAX 所定义），该字符串将被当成 integer 来取值。其它所有情况下都被作为 float 来取值。 

```PHP
<?php
$foo = 1 + "10.5";                // $foo is float (11.5)
$foo = 1 + "-1.3e3";              // $foo is float (-1299)
$foo = 1 + "bob-1.3e3";           // $foo is integer (1)
$foo = 1 + "bob3";                // $foo is integer (1)
$foo = 1 + "10 Small Pigs";       // $foo is integer (11)
$foo = 4 + "10.2 Little Piggies"; // $foo is float (14.2)
$foo = "10.0 pigs " + 1;          // $foo is float (11)
$foo = "10.0 pigs " + 1.0;        // $foo is float (11)     
?> 
```

可以看到字符串中如果有'.'或者e(E)，字符串将会被解析为整数或者浮点数。这些特性在处理Hash字符串时会产生一些安全问题。

```PHP
<?php
"0e132456789"=="0e7124511451155" //true
"0e123456abc"=="0e1dddada"	//false
"0e1abc"=="0"     //true
"0admin" == "0"     //ture

"0x1e240"=="123456"		//true
"0x1e240"==123456		//true
"0x1e240"=="1e240"		//false
?>
```
若Hash字符串以0e开头，在进行!=或者==运算时将会被解析为科学记数法，即0的次方。
或字符串为0x开头，将会被当作十六进制的数。

下面给出几个以0e开头的MD5加密之后的密文。
```
QNKCDZO
0e830400451993494058024219903391
  
s1885207154a
0e509367213418206700842008763514
  
s1836677006a
0e481036490867661113260034900752
  
s155964671a
0e342768416822451524974117254469
  
s1184209335a
0e072485820392773389523109082030
```
下面从具体函数理解弱类型带来的问题。

# 具体函数

## md5()
```PHP
<?php
if (isset($_GET['Username']) && isset($_GET['password'])) {
    $logined = true;
    $Username = $_GET['Username'];
    $password = $_GET['password'];

     if (!ctype_alpha($Username)) {$logined = false;}
     if (!is_numeric($password) ) {$logined = false;}
     if (md5($Username) != md5($password)) {$logined = false;}
     if ($logined){
    echo "successful";
      }else{
           echo "login failed!";
        }
    }
?>
```
要求输入的username为字母，password为数字，并且两个变量的MD5必须一样，这时就可以考虑以0e开头的MD5密文。**md5('240610708') == md5('QNKCDZO')** 可以成功得到flag。

## sha1()
```PHP
<?php

$flag = "flag";

if (isset($_GET['name']) and isset($_GET['password'])) 
{
    if ($_GET['name'] == $_GET['password'])
        echo '<p>Your password can not be your name!</p>';
    else if (sha1($_GET['name']) === sha1($_GET['password']))
      die('Flag: '.$flag);
    else
        echo '<p>Invalid password.</p>';
}
else
    echo '<p>Login first!</p>';
?>
```

sha1函数需要传入的数据类型为字符串类型，如果传入数组类型则会返回NULL

```php
var_dump(sha1([1]));	//NULL
```



因此传入**name[]=1&password[]=2** 即可成功绕过

## strcmp()   php version < 5.3
```PHP
<?php
    $password="***************"
     if(isset($_POST['password'])){

        if (strcmp($_POST['password'], $password) == 0) {
            echo "Right!!!login success";n
            exit();
        } else {
            echo "Wrong password..";
        }
?>
```
和sha1()函数一样，strcmp()是字符串处理函数，如果给strcmp()函数传入数组，无论数据是否相等都会返回0，当然这只存在与PHP版本小于5.3之中。

## intval()
```PHP
<?php
if($_GET[id]) {
   mysql_connect(SAE_MYSQL_HOST_M . ':' . SAE_MYSQL_PORT,SAE_MYSQL_USER,SAE_MYSQL_PASS);
  mysql_select_db(SAE_MYSQL_DB);
  $id = intval($_GET[id]);
  $query = @mysql_fetch_array(mysql_query("select content from ctf2 where id='$id'"));
  if ($_GET[id]==1024) {
      echo "<p>no! try again</p>";
  }
  else{
    echo($query[content]);
  }
}
?>
```
intval()函数获取变量的整数值,程序要求从浏览器获得的id值不为1024，因此输入**1024.1**即可获得flag。

## json_decode()

```PHP
<?php
if (isset($_POST['message'])) {
    $message = json_decode($_POST['message']);
    $key ="*********";
    if ($message->key == $key) {
        echo "flag";
    } 
    else {
        echo "fail";
    }
 }
 else{
     echo "~~~~";
 }
?>
```
需要POST的message中的key值等于程序中的key值，如果传入的key为整数0，那么在比较运算符==的运算下，将会判定字符串为整数0，那么只需要传入数据：**message={"key":0}**

## array_search() 和  in_array()
上面两个函数和前面d额是一样的问题，会存在下面的一些转换为题：
> "admin" == 0; //true
> "1admin" == 1; //true

# 总结
熬夜写文章已经很晚了，就懒得再写总结了。以后每周总结一篇关于代码审计的小知识点。