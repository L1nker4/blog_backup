---
title: Redis底层数据结构-SDS源码分析
date: 2020-05-24 18:59:27
categories:
- Redis
- 源码分析
tags:
- 数据结构
- 源码分析
- Redis
---

# 简介

简单动态字符串(Simple Dynamic Strings)是Redis的基本数据结构之一，主要用于存储字符串和整型数据。SDS兼容C语言标准字符串处理函数，同时保证了二进制安全。

# 数据结构

## 原始版本

在Redis 3.2之前，SDS基本结构如下：
```C
struct {
    //buf中已使用字节数
    int len;
    //buf中剩余字节数
    int free;
    //数据
    char buf[];
}
```

该结构有如下几个优点：
- 有单独的变量存储字符串长度，由于有长度，不会依赖于`\0`终止符，保证二进制安全。
- 字符串存储在buf数组中，兼容C处理字符串的函数。

## 改进
Redis 3.2之后，采用如下结构进行存储。

```C
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

```
sdshdr5中采用位来存储相关信息，其中flags占1字节，其中低3位用来表示type，高5位表示长度，所以长度区间为（0 ~ 31），所以长度大于31的字符串需要采用sdshdr8及以上存储。sdshdr8中flags低3位存储类型，剩余5位闲置。以下是字符串类型的宏定义：
```C
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

### 对齐

GCC支持用`__attribute__`为变量、类型、函数、标签指定特殊属性。这些不是编程语言标准里的内容，而属于编译器对语言的扩展。在声明SDS结构时，采用了`__attribute__ ((__packed__))`，它告诉编译器结构体使用1字节对齐。使用`packed`属性可以节省内存，同时统一多个结构的指针访问。


# 接口

## sdsnewlen

```C
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    //根据字符串长度获取对应的类型
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
     //将SDS_TYPE_5转成SDS_TYPE_5
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    //计算对应结构体头部所需字节数
    int hdrlen = sdsHdrSize(type);
    //指向flags的指针
    unsigned char *fp; /* flags pointer. */
    //分配内存空间
    sh = s_malloc(hdrlen+initlen+1);
    //判断是否为"SDS_NOINIT"
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        //初始化结构体空间数据
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    //s是指向buf的指针
    s = (char*)sh+hdrlen;
    //指向flags
    fp = ((unsigned char*)s)-1;
    //针对不同结构体类型进行初始化操作
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    //如果两个参数不为0，将字符串数据复制到buf数组
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

## sdsfree

通过定位SDS的头部，调用`s_free`释放内存。为了减少申请内存的开销，SDS可以提供`sdsclear`进行重置达到清空的目的。
```C
/* Free an sds string. No operation is performed if 's' is NULL. */
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
```

## sdsclear

该方法将字符串长度设置为0，同时将数组第一个元素置为终止符。
```C
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}
```

## sdscatsds

通过调用`sdscatlen`实现拼接逻辑。
```C
sds sdscatsds(sds s, const sds t) {
    return sdscatlen(s, t, sdslen(t));
}

sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);
    //对buf进行扩容，会进行扩容检测
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    //调用memcpy实现拼接
    memcpy(s+curlen, t, len);
    //更新长度
    sdssetlen(s, curlen+len);
    //设置终止符
    s[curlen+len] = '\0';
    return s;
}
```

## sdsMakeRoomFor

SDS扩容操作：
```C
#define SDS_MAX_PREALLOC (1024*1024)

sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    //剩余有效空间
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    //判断有效空间是否大于需要增加的空间大小
    if (avail >= addlen) return s;
    //sds已使用长度
    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    //新长度
    newlen = (len+addlen);
    //define SDS_MAX_PREALLOC (1024*1024)
    //如果新长度 < 1MB，扩大两倍
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        //否则扩大1MB
        newlen += SDS_MAX_PREALLOC;

    //根据新长度获取对应type
    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
    //
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        //无需更改type，通过s_realloc扩大buf数组即可
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
         //按新的数组长度申请内存
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        //将buf移动到新位置
        memcpy((char*)newsh+hdrlen, s, len+1);
        //释放原来的指针
        s_free(sh);
        //s是指向buf起始位置的指针
        s = (char*)newsh+hdrlen;
        //赋值flags
        s[-1] = type;
        //设置新的字符串长度
        sdssetlen(s, len);
    }
    //设置新的数组长度
    sdssetalloc(s, newlen);
    return s;
}
```

## sdsRemoveFreeSpace

缩容操作：

```C
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    //字符串长度
    size_t len = sdslen(s);
    //剩余有效长度
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    如果不需要缩容直接返回
    if (avail == 0) return s;

    //根据长度获取类型
    type = sdsReqType(len);
    //获取头部长度
    hdrlen = sdsHdrSize(type);

    /* If the type is the same, or at least a large enough type is still
     * required, we just realloc(), letting the allocator to do the copy
     * only if really needed. Otherwise if the change is huge, we manually
     * reallocate the string to use the different header type. */
     //如果原始类型和更新类型相似，或者type > SDS_TYPE_8
    if (oldtype==type || type > SDS_TYPE_8) {
    //重分配
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
    //申请内存空间
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        //复制
        memcpy((char*)newsh+hdrlen, s, len+1);
        //释放指针
        s_free(sh);
        s = (char*)newsh+hdrlen;
        //设置类型
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```

# 总结
本文主要对Redis中的简单动态字符串的数据结构与基本API实现做了简要分析，基本了解了SDS如何保证二进制安全与SDS的缩容扩容策略的实现。