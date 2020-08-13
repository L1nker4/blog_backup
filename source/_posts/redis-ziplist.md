---
title: Redis底层数据结构-ZipList源码分析
date: 2020-05-26 16:58:29
categories:
- Redis
- 源码分析
tags:
- 数据结构
- 源码分析
- Redis
---

# 简介
压缩列表（ziplist）的本质是一个字节数组，主要是Redis为了节省内存而设计的数据结构。在Redis的有序集合和散列表都使用了ziplist，当有序集合或散列表的元素个数比较少，并且元素都是短字符串时，使用ziplist作为其底层数据结构。

压缩列表的基本结构基本如下所示：
```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```

- uint32_t zlbytes：压缩列表的字节长度，占4个字节
- uint32_t zltail：压缩列表尾元素相对于起始地址的偏移量，占4个字节，方便从列表尾部进行操作
- uint16_t zllen：元素个数，占2个字节，元素个数无法超过2^16-1，只能通过遍历整个列表才能获取到个数
- uint8_t zlend：列表的结尾元素，占1个字节，值为255（0xff）
- entry：列表的元素，可以是字节数组或者整数
    - prevlen：表示前一个元素的字节长度，1~5个字节表示，当前一个元素长度小于254字节，用1个字节表示，大于或等于254字节时，用5个字节表示，该情况下，第一个字节为`0xFE`，剩余4个字节表示真正长度
    - encoding：表示当前元素的编码，编码表示当前存储的是字节数组还是整数
    - entry：存储数据内容

encoding选项：
```C
#define ZIP_STR_06B (0 << 6)
#define ZIP_STR_14B (1 << 6)
#define ZIP_STR_32B (2 << 6)
#define ZIP_INT_16B (0xc0 | 0<<4)
#define ZIP_INT_32B (0xc0 | 1<<4)
#define ZIP_INT_64B (0xc0 | 2<<4)
#define ZIP_INT_24B (0xc0 | 3<<4)
#define ZIP_INT_8B 0xfe
```
下面是一个包含两个元素的ziplist，存储的数据为字符串“2”和“5”。它由15个字节组成
```
   [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
         |             |          |       |       |     |
      zlbytes        zltail    entries   "2"     "5"   end
```

Redis通过宏定义来对以上部分进行快速定位，zl为压缩列表首地址指针。
```C

#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))

#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))

#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))

#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

#define ZIPLIST_ENTRY_HEAD(zl)  ((zl)+ZIPLIST_HEADER_SIZE)

#define ZIPLIST_ENTRY_TAIL(zl)  ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))

#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-1)
```

# 数据结构
压缩列表在获取元素长度、获取元素内容都需要经过解码运算，解码后的数据结果存储在zlentry结构中。
```C
typedef struct zlentry {
    //上述的prevlen的长度
    unsigned int prevrawlensize;
    //上述的prevlen的内容
    unsigned int prevrawlen;
    //encoding字段的长度
    unsigned int lensize;
    //元素数据的长度
    unsigned int len;
    //当前元素的首部长度，prevrawlensize + lensize
    unsigned int headersize;
    //数据类型
    unsigned char encoding;
    //指向元素的首地址的指针
    unsigned char *p;           
} zlentry;
```

# 接口

## ziplistNew

创建压缩列表，返回值为压缩列表的首地址。
```C
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))

unsigned char *ziplistNew(void) {
    //计算分配的字节数
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    //申请内存空间
    unsigned char *zl = zmalloc(bytes);
    //初始化
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    //设置结尾标识符0xff
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

## ziplistInsert

该接口在指定的位置p，插入的数据的地址指针为s，长度为slen。新插入的数据占据p原来的位置，p之后的数据项向后移动。
```C
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    return __ziplistInsert(zl,p,s,slen);
}

unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    //初始化变量
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789;
    zlentry tail;

    //如果p不是尾元素
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    //尝试将数据内容解析为整数
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        //按整数类型编码进行存储
        reqlen = zipIntSize(encoding);
    } else {
        //按字节数组类型编码进行存储
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    //不是从尾部进行插入时，需要确保下一个entry可以存储当前entry的长度
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    //存储偏移量
    offset = p-zl;
    //调用realloc重新分配空间
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    //重新偏移到插入位置p
    p = zl+offset;

    //数据复制
    if (p[0] != ZIP_END) {
        //因为zlend恒为0xff，所以减一
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        //更新entryX + 1元素的prevlen数据
        if (forcelarge)
        //该entry的prevlen仍然用5个字节存储时
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        //更新zltail
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        //
        zipEntry(p+reqlen, &tail);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
       
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    //写entry
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    //更新zllen
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```


## ziplistDelete
zl指向压缩列表首地址，*p指向需要删除的entry的首地址，返回值为压缩列表首地址。`__ziplistDelete`可以同时删除多个连续元素，输入参数p指向的是首个待删除元素的地址，num表示待删除的元素个数。
```C
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p) {
    size_t offset = *p-zl;
    zl = __ziplistDelete(zl,*p,1);
    
    *p = zl+offset;
    return zl;
}

unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    zlentry first, tail;
    //解码第一个待删除的元素
    zipEntry(p, &first);
    //遍历所有待删除的元素，同时指针p向后偏移
    for (i = 0; p[0] != ZIP_END && i < num; i++) {
        p += zipRawEntryLength(p);
        deleted++;
    }
    //该变量为待删除元素总长度
    totlen = p-first.p; /* Bytes taken by the element(s) to delete. */
    if (totlen > 0) {
        if (p[0] != ZIP_END) {
            //计算元素entryN长度的变化量
            nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);

            //更新元素entryN的prevlen数据
            p -= nextdiff;
            zipStorePrevEntryLength(p,first.prevrawlen);

            //更新zltail
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen);
            zipEntry(p, &tail);
            if (p[tail.headersize+tail.len] != ZIP_END) {
                ZIPLIST_TAIL_OFFSET(zl) =
                   intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
            }

            //数据复制
            memmove(first.p,p,
                intrev32ifbe(ZIPLIST_BYTES(zl))-(p-zl)-1);
        } else {
            /* The entire tail was deleted. No need to move memory. */
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe((first.p-zl)-first.prevrawlen);
        }

        //重新分配空间，与添加逻辑相似
        offset = first.p-zl;
        zl = ziplistResize(zl, intrev32ifbe(ZIPLIST_BYTES(zl))-totlen+nextdiff);
        ZIPLIST_INCR_LENGTH(zl,-deleted);
        p = zl+offset;

        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }
    return zl;
}
```

# 总结
本文主要对压缩列表的基本概念与在Redis中的具体实现做了简要分析，同时对压缩列表在Redis中的添加删除接口做了简要的源代码分析。使用压缩列表能有效地节省内存，在Redis的Hash结构中，当field比较少时，采用压缩列表进行存储，当达到对应阈值，转为dict进行存储，与JDK1.8中的HashMap的策略有异曲同工之妙。