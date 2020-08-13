---
title: PHP数组整数键名截断问题
date: 2018-11-23 17:02:35
categories:
- Web安全
- CTF
tags:
- 安全
- PHP代码审计
- CTF
---

<!--more-->

# 前言

这段时间在做CHY师傅整理的CTF代码审计题目，遇到较难或者比较有意思的题目，都会记下笔记，这次分享一篇关于PHP处理数组时的一个漏洞，这里给出CHY师傅的题目地址：
> [https://github.com/CHYbeta/Code-Audit-Challenges/](https://github.com/CHYbeta/Code-Audit-Challenges/)


# 分析
首先看看PHP官方对这个错误的介绍：
> [https://bugs.php.net/bug.php?id=69892](https://bugs.php.net/bug.php?id=69892)


>var_dump([0 => 0] === [0x100000000 => 0]); // bool(true)

下面来看代码：
```PHP
<?php

/*******************************************************************
 * PHP Challenge 2015
 *******************************************************************
 * Why leave all the fun to the XSS crowd?
 *
 * Do you know PHP?
 * And are you up to date with all its latest peculiarities?
 *
 * Are you sure?
 *
 * If you believe you do then solve this challenge and create an
 * input that will make the following code believe you are the ADMIN.
 * Becoming any other user is not good enough, but a first step.
 *
 * Attention this code is installed on a Mac OS X 10.9 system
 * that is running PHP 5.4.30 !!!
 *
 * TIPS: OS X is mentioned because OS X never runs latest PHP
 *       Challenge will not work with latest PHP
 *       Also challenge will only work on 64bit systems
 *       To solve challenge you need to combine what a normal
 *       attacker would do when he sees this code with knowledge
 *       about latest known PHP quirks
 *       And you cannot bruteforce the admin password directly.
 *       To give you an idea - first half is:
 *          orewgfpeowöfgphewoöfeiuwgöpuerhjwfiuvuger
 *
 * If you know the answer please submit it to info@sektioneins.de
 ********************************************************************/

$users = array(
        "0:9b5c3d2b64b8f74e56edec71462bd97a" ,
        "1:4eb5fb1501102508a86971773849d266",
        "2:facabd94d57fc9f1e655ef9ce891e86e",
        "3:ce3924f011fe323df3a6a95222b0c909",
        "4:7f6618422e6a7ca2e939bd83abde402c",
        "5:06e2b745f3124f7d670f78eabaa94809",
        "6:8e39a6e40900bb0824a8e150c0d0d59f",
        "7:d035e1a80bbb377ce1edce42728849f2",
        "8:0927d64a71a9d0078c274fc5f4f10821",
        "9:e2e23d64a642ee82c7a270c6c76df142",
        "10:70298593dd7ada576aff61b6750b9118"
);

$valid_user = false;

$input = $_COOKIE['user'];
$input[1] = md5($input[1]);

foreach ($users as $user)
{
        $user = explode(":", $user);
        if ($input === $user) {
                $uid = $input[0] + 0;
                $valid_user = true;
        }
}

if (!$valid_user) {
        die("not a valid user\n");
}

if ($uid == 0) {

        echo "Hello Admin How can I serve you today?\n";
        echo "SECRETS ....\n";

} else {
        echo "Welcome back user\n";
}
?>

```
首先来看看代码逻辑：
> * 从**cookie**中获取**user**
> * 遍历定义的**users**数组，判断传入的数据**input**是否等于它

这里我们要绕过两个条件判断，经过md5解密网站的测试，发现只能解出这一条。
> 06e2b745f3124f7d670f78eabaa94809      //hund

初步可以判断传入的数据为：**Cookie: user[0]=5;user[1]=hund;**
这样传入数据，就可以成功绕过第一个判断，接下来只要让**uid**为0即可。
根据前面给出的数组处理漏洞，分析一下PHP源码**php-src/Zend/zend_hash.c**
```C
//php5.2.14
ZEND_API int zend_hash_compare(HashTable *ht1, HashTable *ht2, compare_func_t compar, zend_bool ordered TSRMLS_DC)
{
	Bucket *p1, *p2 = NULL;
	int result;
	void *pData2;

	IS_CONSISTENT(ht1);
	IS_CONSISTENT(ht2);

	HASH_PROTECT_RECURSION(ht1); 
	HASH_PROTECT_RECURSION(ht2); 

	result = ht1->nNumOfElements - ht2->nNumOfElements;
	if (result!=0) {
		HASH_UNPROTECT_RECURSION(ht1); 
		HASH_UNPROTECT_RECURSION(ht2); 
		return result;
	}

	p1 = ht1->pListHead;
	if (ordered) {
		p2 = ht2->pListHead;
	}

	while (p1) {
		if (ordered && !p2) {
			HASH_UNPROTECT_RECURSION(ht1); 
			HASH_UNPROTECT_RECURSION(ht2); 
			return 1; /* That's not supposed to happen */
		}
		if (ordered) {
			if (p1->nKeyLength==0 && p2->nKeyLength==0) { /* numeric indices */
				result = p1->h - p2->h;
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
			} else { /* string indices */
				result = p1->nKeyLength - p2->nKeyLength;
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
				result = memcmp(p1->arKey, p2->arKey, p1->nKeyLength);
				if (result!=0) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return result;
				}
			}
			pData2 = p2->pData;
		} else {
			if (p1->nKeyLength==0) { /* numeric index */
				if (zend_hash_index_find(ht2, p1->h, &pData2)==FAILURE) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return 1;
				}
			} else { /* string index */
				if (zend_hash_quick_find(ht2, p1->arKey, p1->nKeyLength, p1->h, &pData2)==FAILURE) {
					HASH_UNPROTECT_RECURSION(ht1); 
					HASH_UNPROTECT_RECURSION(ht2); 
					return 1;
				}
			}
		}
		result = compar(p1->pData, pData2 TSRMLS_CC);
		if (result!=0) {
			HASH_UNPROTECT_RECURSION(ht1); 
			HASH_UNPROTECT_RECURSION(ht2); 
			return result;
		}
		p1 = p1->pListNext;
		if (ordered) {
			p2 = p2->pListNext;
		}
	}
	
	HASH_UNPROTECT_RECURSION(ht1); 
	HASH_UNPROTECT_RECURSION(ht2); 
	return 0;
}
```


bucket结构体：
```C
typedef struct bucket {
	ulong h;						/* Used for numeric indexing */
	uint nKeyLength;
	void *pData;
	void *pDataPtr;
	struct bucket *pListNext;
	struct bucket *pListLast;
	struct bucket *pNext;
	struct bucket *pLast;
	char arKey[1]; /* Must be last element */
} Bucket;
```
关键点就在**result = p1->h - p2->h**，在**unsigned long**转为**int**时会少掉4个字节。
4294967296转为int就为0。
最终payload：
> **Cookie: user[4294967296]=5;user[1]=hund;**


# 总结

不断的前进