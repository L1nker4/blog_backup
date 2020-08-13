---
title: 南京邮电大学CTF平台Web系列WriteUp
date: 2018-06-25 14:10:49
categories:
- Web安全
- CTF
tags:
- CTF
---
<!--more-->

## 签到1

直接查看源代码，得到flag
nctf{flag_admiaanaaaaaaaaaaa}

## 签到2
要求输入zhimakaimen
审查元素更改输入框type为text 长度改为11
提交得到nctf{follow_me_to_exploit}

## 这题不是Web
这题真的不是Web
下载图片，以文本方式打开，最后一行即为flag

## 层层递进
burpsuite抓包之后，扫出一个404.html
源代码如下
<!-- more -->
``` html

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<HTML><HEAD><TITLE>有人偷偷先做题，哈哈飞了吧？</TITLE>
<META HTTP-EQUIV="Content-Type" Content="text/html; charset=GB2312">
<STYLE type="text/css">
  BODY { font: 9pt/12pt 宋体 }
  H1 { font: 12pt/15pt 宋体 }
  H2 { font: 9pt/12pt 宋体 }
  A:link { color: red }
  A:visited { color: maroon }
</STYLE>
</HEAD><BODY>
<center>
<TABLE width=500 border=0 cellspacing=10><TR><TD>
<!-- Placed at the end of the document so the pages load faster -->
<!--  
<script src="./js/jquery-n.7.2.min.js"></script>
<script src="./js/jquery-c.7.2.min.js"></script>
<script src="./js/jquery-t.7.2.min.js"></script>
<script src="./js/jquery-f.7.2.min.js"></script>
<script src="./js/jquery-{.7.2.min.js"></script>
<script src="./js/jquery-t.7.2.min.js"></script>
<script src="./js/jquery-h.7.2.min.js"></script>
<script src="./js/jquery-i.7.2.min.js"></script>
<script src="./js/jquery-s.7.2.min.js"></script>
<script src="./js/jquery-_.7.2.min.js"></script>
<script src="./js/jquery-i.7.2.min.js"></script>
<script src="./js/jquery-s.7.2.min.js"></script>
<script src="./js/jquery-_.7.2.min.js"></script>
<script src="./js/jquery-a.7.2.min.js"></script>
<script src="./js/jquery-_.7.2.min.js"></script>
<script src="./js/jquery-f.7.2.min.js"></script>
<script src="./js/jquery-l.7.2.min.js"></script>
<script src="./js/jquery-4.7.2.min.js"></script>
<script src="./js/jquery-g.7.2.min.js"></script>
<script src="./js/jquery-}.7.2.min.js"></script>
-->

<p>来来来，听我讲个故事：</p>
<ul>
<li>从前，我是一个好女孩，我喜欢上了一个男孩小A。</li>
<li>有一天，我终于决定要和他表白了！话到嘴边，鼓起勇气...
</li>
<li>可是我却又害怕的<a href="javascript:history.back(1)">后退</a>了。。。</li>
</ul>
<h2>为什么？<br>为什么我这么懦弱？</h2>
<hr>
<p>最后，他居然向我表白了，好开森...说只要骗足够多的笨蛋来这里听这个蠢故事浪费时间，</p>
<p>他就同意和我交往！</p>
<p>谢谢你给出的一份支持！哇哈哈\(^o^)/~！</p>

</TD></TR></TABLE>
</center>
</BODY></HTML>
```
注意引入的js文件名

## 单身二十年
burpsuite爬行出一个search_key.php，flag藏在Response里面

## MYSQL
打开首先提示robots.txt，打开发现以下代码

``` php
TIP:sql.php

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
半吊子PHP水平分析一波，intval()函数是将变量转化为整数，需要得到的id变量为1024，但不允许输入的值为1024，因为有intval()函数，可以输入1024.1a，从而得到flag。

## COOKIE
TIP: 0==not
给出的提示以上信息，题目又是cookie，查看cookie发现它的值为0，将它改成1，得到flag:nctf{cookie_is_different_from_session}
![nctf](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/nctf1.png)

