---
title: 浅析文件上传漏洞
date: 2018-10-01 21:05:56
categories:
- Web安全
- CTF
tags:
- PHP代码审计
- 安全
- CTF
---

<!--more-->

# 0x01简介 #

文件上传漏洞简而言之就是攻击者可以通过某些手段上传非法文件到服务器端的一种漏洞，攻击者往往会以此得到webshell，从而进一步提权。为此开发者也会想出各种方法去防止上传漏洞的产生。以下列出几种常见的校验方式：
- 前端JS校验
- content-type字段校验
- 服务器端后缀名校验
- 文件头校验
- 服务器端扩展名校验
  

  下面从几个实例来详细解释上传漏洞的原理与突破方法。

# 0x02正文 #

### 一、前端JS校验 ###
JS校验往往只是通过脚本获得文件的后缀名，再通过白名单验证，以下列出JS代码供参考：
```JavaScript
<script type="text/javascript">
    function checkFile() {
        var file = document.getElementsByName('upload_file')[0].value;
        if (file == null || file == "") {
            alert("请选择要上传的文件!");
            return false;
        }
        //定义允许上传的文件类型
        var allow_ext = ".jpg|.png|.gif";
        //提取上传文件的类型
        var ext_name = file.substring(file.lastIndexOf("."));
        //判断上传文件类型是否允许上传
        if (allow_ext.indexOf(ext_name) == -1) {
            var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件,当前文件类型为：" + ext_name;
            alert(errMsg);
            return false;
        }
    }
</script>
```
此种校验方式往往很容易突破，一种是通过抓取HTTP数据包，修改重放便可以突破。或者通过浏览器修改前端代码，从而改变代码逻辑。

### 二、content-type字段校验 ###
首先对MIME进行简单的知识普及
>http://www.cnblogs.com/jsean/articles/1610265.html

>https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_Types

```php
<?php
        if($_FILES['userfile']['type'] != "image/gif")  #这里对上传的文件类型进行判断，如果不是image/gif类型便返回错误。
                {   
                 echo "Sorry, we only allow uploading GIF images";
                 exit;
                 }
         $uploaddir = 'uploads/';
         $uploadfile = $uploaddir . basename($_FILES['userfile']['name']);
         if (move_uploaded_file($_FILES['userfile']['tmp_name'], $uploadfile))
             {
                 echo "File is valid, and was successfully uploaded.\n";
                } else {
                     echo "File uploading failed.\n";
    }
     ?>
```
此类校验方式绕过也十分简单，burpsuite截取数据包，修改content-type值，发送数据包即可。

### 三、文件头校验 ###

文件头基本概念：
>https://www.cnblogs.com/mq0036/p/3912355.html

上传webshell的过程中,添加文件头便可以突破此类校验。例如
```php
GIF89a<?php phpinfo(); ?>
```

### 四、服务器端扩展名校验 ###
此类校验的上传绕过往往较为复杂，常常与其他漏洞搭配使用，如解析漏洞，文件包含漏洞。

```php
<?php
$type = array("php","php3");
//判断上传文件类型
$fileext = fileext($_FILE['file']['name']);
if(!in_array($fileext,$type)){
    echo "upload success!";
}
else{
    echo "sorry";
}
?>
```
Apache解析漏洞，IIS6.0解析漏洞在此不再过多阐述。

### 五、编辑器上传漏洞 ###
早期编辑器例如FCK,eWeb编辑器正在逐渐淡出市场，但是并没有使富文本编辑器的占用率变低，因此在渗透测试的过程中，一般可以查找编辑器漏洞从而获得webshell。

# 0x03总结 #
在渗透测试的过程中,遇到上传点，通常情况下截取数据包，再通过各种姿势进行bypass。关于防护的几点建议：

- 在服务器端进行校验
- 上传的文件可以进行时间戳md5进行命名
- 上传的路径隐藏
-  文件存储目录权限控制

当然最重要的是，维持网站中间件的安全，杜绝旧版解析漏洞的存在。

