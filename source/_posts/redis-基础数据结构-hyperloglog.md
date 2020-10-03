---
title: redis-基础数据结构: HyperLogLog（二）源码与设计
date: 2020-10-02 11:26:44
tags:
    - redis
---

[上一篇文章](其实还没有开始写.com)介绍了 HyperLogLog 的工作原理，这里介绍一下 redis 中的具体实现

## 从伯努利实验到UV计数

从伯努利实验中我们可以通过连续反面的次数来估算抛掷次数，而 HyperLogLog 正是利用的这个原理来实现诸如UV计数等功能。那么如何将UV计数和伯努利实验联系起来呢？HLL 适用于字符串，redis 将其哈希，得到二进制哈希值的0、1就可以和硬币的正反面对应；通过统计其哈希值连续 0 的个数从而对应连续出现反面的次数

所以按照伯努利实验的思路，我们只需要一个计数器，记录下当前哈希值，每次计算添加元素的哈希值的最低位开始的0的个数，如果比当前的大，就更新这个计数器。但是，从伯努利实验中我们也看到只抛一轮实验计算得到的结果误差是比较大的，所以会抛多轮并计算其平均值（或者加权平均）进行优化。HLL 也是类似，它用了 16384 个计数器，哈希值的低14位用于确定计数器，高50位用于计算连续0的个数，从而达到平均的效果。

当使用`PFADD`命令添加一个元素时，首先将该元素哈希为一个 uint64，并且通过哈希值的低14bit定位到它所在的 register，然后计算该哈希值从15位开始连续为 0 的 bit 的个数，如果新元素的连续 0bit 个数大于该 register，那么就更新 register。

下面这个函数用于确定 register 的下标并计算连续为 0 的 bit 的个数：

```c
int hllPatLen(unsigned char *ele, size_t elesize, long *regp) {
    uint64_t hash, bit, index;
    int count;

    hash = MurmurHash64A(ele,elesize,0xadc83b19ULL);
    index = hash & HLL_P_MASK;      /* Register index. */
    hash >>= HLL_P;                 /* Remove bits used to address the register. */
    hash |= ((uint64_t)1<<HLL_Q);   /* Make sure the loop terminates and count will be <= Q+1. */
    bit = 1;
    count = 1;                      /* Initialized to 1 since we count the "00000...1" pattern. */
    while((hash & bit) == 0) {
        count++;
        bit <<= 1;
    }
    *regp = (int) index;
    return count;
}
```

这里有几点需要注意：

- 终结连续 0bit 串的 1 也需要被包含进来，比如 001，其计数值为 3；
-

##  数据结构

hll 底层使用的是 sds 存储数据，并且通过一个 header 来管理数据：

```c
struct hllhdr {
    char     magic[4];      /* "HYLL" */
    uint8_t  encoding;      /* HLL_DENSE or HLL_SPARSE. */
    uint8_t  notused[3];    /* Reserved for future use, must be zero. */
    uint8_t  card[8];       /* Cached cardinality, little endian. */
    uint8_t  registers[];   /* Data bytes. */
}
```

- magic：即 H、Y、L、L 四个字母的 ascii 值，用以和其他 sds 区别
- encoding：针对不同的场景，redis 会采用不同的编码方式，目前一共有 3 种编码方式
- card：即 cardinality，用以缓存数目，最高位为 1 表示缓存是有效的，此时`pfcount`命令直接拿这个数值，否则就需要重新计算
- registers：存储着所有的计数器

`hllhdr`中的`registers`柔性数组一共有 16384 个 register，每个 register 占 6bit。为什么是 6bit 呢？因为一个`int64`最多只有 64 个 bit 为 0，用 6bit 就可以表示了。

## 三种编码方式

按照`hllhdr`的布局，一个 HLL 初始条件下就得占用 4+1+3+8+(16384*6/8)= 12304byte，也就是差不多 12KB，在 redis 中这被称为 dense 编码；而很多时候我们计数可能没有那么多，其中很多 register 都是 0，这个时候也占用 12KB 有些浪费；所以对于大部分 register 为 0 的情况，redis 提出了 sparse 编码；此外，在`PFMERGE`命令的实现中，还使用到了另外一种编码方式。

### sparse 编码

既然大多数 register 的值都为 0，那么可以想象，我们不直接存储 0 值，而是存储为 0 的 register 的个数，就可以达到节省内存的目的。而事实上 redis 也的确是这么做的，并且实现了更加精细的粒度控制，

对于 sparse 编码，redis 存储的是三种操作码：

- ZERO: 用 00xxxxxx 来表示，其中 xxxxxx 表示连续为 0 的 register 的个数
- XZERO: 用 01xxxxxx yyyyyyyy 双字节来表示连续为 0 的 register 的个数，XZERO 只能表示 register <= 64 的情况，而 XZERO 则可以表示更多
- VAL: 用 1vvvvvvxx 来表示连续多个 register 的值相等情况，其中 vvvvv 表示的是 register 的值，xx 表示的是 register 的个数

有几点需要注意：

- 其实 opcode 表示的 register 的真实数目要在其数值的基础上 +1，因为 0个 register 其实是没有意义的
- XZERO 中的两个字节，xxxxxx 是高位，yyyyyyyy 是低位

```c
#define HLL_SPARSE_ZERO_LEN(p)  (((*(p)) & 0x3f)+1)
#define HLL_SPARSE_XZERO_LEN(p) (((((*(p)) & 0x3f) << 8) | (*((p)+1)))+1)
#define HLL_SPARSE_VAL_LEN(p)   (((*(p)) & 0x3)+1)
```

为什么这样设计呢？

- 不同的 opcode，需要使用若干 bit 来进行区分定义
- 首先要考虑到创建 hll 时 16384 个 register 都为 0 的情况，这样就至少需要 14 个 bit，也就是说 2byte
- 用两个字节来表示 0 比较浪费，可能存在少量连续 register 为 0 的情况，这个可以用 1 byte 来表示
- 需要考虑到连续多个 register 为同一数值（不一定是 0）的情况

这样有哪些限制呢？

- 可以看到 VAL 指令能表示的 register 的值最大为 32，所以 sparse 形态的 hyperloglog 也要求所有 register 值最大为 32，否则就需要转为 dense 形态
- VAL 能表示的具有相同值的 register 的个数最多为 4，这我感觉也是经过考量的，因为用 2bit 表示 3 种类型，所以有一种类型可以用一个 bit 表示即可，多出来的 bit 可以作为计数使用（比如 1vvvvvxxx 这样可以最多以及最大表示连续 8 个值为 16 的 register

#### sparse 升级到 dense
在两种情况下，稀疏形态会被转换为稠密形态：

- 某个 register 的值大于 32
- 总的 hyperloglog 的大小大于`server.hll_sparse_max_bytes`配置项的值

```c
int hllSparseToDense(robj *o) {
    sds sparse = o->ptr, dense;
    struct hllhdr *hdr, *oldhdr = (struct hllhdr*) sparse;
    int idx = 0, runlen, regval;
    uint8_t *p = (uint8_t*)sparse, *end = p+sdslen(sparse);

    dense = sdsnewlen(NULL,HLL_DENSE_SIZE);
    hdr = (struct hllhdr *) dense;
    *hdr = *oldhdr; // 拷贝原来 header 中的 magic 和 card
    hdr->encoding = HLL_DENSE;

    p += HLL_HDR_SIZE;
    while (p <= end) {
        if (HLL_SPARSE_IS_ZERO(p)) {
            runlen = HLL_SPARSE_ZERO_LEN(p)
            idx += runlen;
            p++;
        } else if (HLL_SPARSE_IS_XZERO) {
            runlen = HLL_SPARSE_XZERO_LEN(p);
            idx += runlen;
            p += 2;
        } else {
            runlen = HLL_SPARSE_VAL_LEN(p);
            regval = HLL_SPARSE_VAL_VALUE(p);
            if ((runlen + idx) > HLL_REGISTERS) break;
            while (runlen--) {
                HLL_DENSE_SET_REGISTER(hdr->registers,idx,regval);
                idx++;
            }
            p++;
        }
    }

    if (idx != HLL_REGISTERS) {
    	sdsfree(dense);
        return C_ERROR;
    }

    sdsfree(sparse);
    o->ptr = dense;
    return C_OK;
}
```

代码比较简单，主要是注意错误处理，防止内存越界，其中类似的 for 循环在 sparse 编码相关的多个函数中都使用到了。

#### sparse 编码下如何存取数据

和 dense 编码一样，sparse 编码也有存取数据的操作，其中 set 操作比较复杂，需要仔细理解：

首先是检查 set 的值是否超出了 32，这是`VAL`类型的 opcode 所能存储的最大数值，超出了就得升级为 dense 编码。没有超出的话，那么就得尝试着往 sparse 里面 set 数值，而在 sparse 中进行 set 操作，可能会造成 opcode 的分裂，所以需要为分裂预留空间，最大的分裂情况是`XZERO`类型的 opcode 分裂为 `XZERO|VAL|XZERO`，增加了 3byte，所以先预分配 3byte。此外分裂可能造成 sparse 的大小超出了阈值从而导致 promote，不过得先尝试着 set 才知道会不会 promote：

然后定位到第 index 个 register 所在的 opcode，以及前一个和后一个 opcode。这里和前面 for 循环逻辑类似，

```c 
    while (p < end) {
        long oplen;

        oplen = 1;
        if (HLL_SPARSE_IS_ZERO(p)) {
            span = HLL_SPARSE_ZERO_LEN(p);
        } else if (HLL_SPARSE_IS_XZERO(p)) {
            span = HLL_SPARSE_XZERO_LEN(p)
        } else {
            span = HLL_SPARSE_VAL_LEN(p);
            oplen = 2;
        }

        if (index < fist+span-1) break;
        prev = p;
        p += oplen;
        first += span;
    }

    if (span == 0 || p >= end) return -1;

    next = HLL_SPARSE_IS_ZERO(p) ? p+2 : p+1;
    if (next >= end) next = NULL;
```

> 我感觉这样的循环是最难控制的，以及如何确定循环结束时各个变量的状态，原来看《算法导论》的时候学到"循环不变式"方法用的也不是很顺手

当循环结束，`p`指向第 index 个 register 所在的 opcode，并且由`is_zero`、`is_xzero`和`is_val`指明该 opcode 的类型，first 指向的是该 opcode 包含的第一个 register 的下标，next 和 prev 分别表示前一个和后一个 opcode（当`p`是第一个或者最后一个 opcode 的话，对于的`prev`和`next`可能为`NULL`）

随后 redis 处理了几种比较特殊但是不需要分裂、容易处理的情况:

1. p 是一个 VAL 类型的 opcode，而且其值 >= count，此时无需更新
2. p 是一个 VAL 类型的 opcode，长度为 1， 且其值 < count，此时更新其值即可
3. p 是一个长度为 1 的 ZERO 类型的 opcode，此时只需要将其类型更新为 VAL，并且设置其值为 count 即可

这里有一点需要注意，对于 case2 和 case3，更新 VAL 之后，p 和其左右的 opcode 可能是可以合并的，redis 会在 updated 中做这个优化。

如果以上的几种特殊情况都不满足，那么就必须分裂 opcode，而 opcode 被分裂后最长有 5byte，所以 redis 将分裂后的数据存储在一个长度为 5byte 的临时缓存区内，然后将原来的内存缓冲区腾出合适大小的空间，并把临时缓冲区中的数据拷贝进去；举一个 VAL 类型的 opcode 分裂的情况：

```c 
    uint8_t seq[5]; *n = seq;
    int last = first+span-1;
    int len;
    if (is_zero || is_xzero) {
        ...
    } else {
        curval = HLL_SPARSE_VAL_VALUE(p);
        if (index != first) {
            len = index-first;
            HLL_SPARSE_VAL_SET(n,curval,len);
            n++
        }
        HLL_SPARSE_VAL_SET(n,count,1);
        n++;
        if (index != last) {
            len = last-index;
            HLL_SPARSE_VAL_SET(n,curval,len);
            n++
        }
    }

    int seqlen = n-seq;
    int oldlen = is_xzero ? 2 : 1;
    int deltalen = oldlen - len;

    if (deltalen > 0 &&
        sdslen(o->ptr)+delta > server.hll_sparse_max_bytes) goto promote;
    if (deltalen && next) memmove(next+deltalen, next, end-next);
    sdsIncrLen(o->ptr, deltalen);
    memcpy(p,seq,seqlen);
    end += deltalen;
```

需要注意的是此时仍可以优化：

- VAL 类型的 opcode 分裂，导致分裂出去的左右两个 opcode 可以和其相邻的 opcode 合并
- opcode 从长度为 1 的 ZERO 更新为长度为 1 的 VAL，从而可以和其相邻 opcode 合并
- opcode 从长度为 1 的 VAL 更新为长度仍为 1，但是值更大的 VAL，从而可以与其相邻的 opcode 合并

![redis-hll-merge-adjacent-opcode](images/inpost/redis/hll/redis-hll-merge-adjacent-opcode.png)



```c
updated:
    p = prev ? prev : sparse;
    int scanlen = 5;
    while (p < end && scanlen--) {
        if (HLL_SPARSE_IS_XZERO(p)) {
            p += 2;
            continue
        } else if (HLL_SPARSE_IS_ZERO(p)) {
            p++;
            continue
        }

        if (p+1 < end &&& HLL_SPARSE_IS_VAL(p+1)) {
            int v1 = HLL_SPARSE_VAL_VALUE(p);
            int v2 = HLL_SPARSE_VAL_VALUE(p+1);
            if (v1 == v2) {
            	int len = HLL_SPARSE_VAL_LEN(p)+HLL_SPARSE_VAL_LEN(p+1)
                if (len <= HLL_SPARSE_VAL_MAX_LEN) {
                    HLL_SPARSE_VAL_SET(p+1,v1,len);
                    // QUESTION: 为什么不是 memmove(p,p+1,end-p-1)
                    memmove(p,p+1,end-p)
                    sdsIncrLen(o->ptr,-1);
                    end--;
                    continue;
                }
            }
        }
        p++;
    }
```

大致思路：

- 如果 p 和 p+1 可以合并，那么将 p+1 的 value 置为原来 p 和 p+1 的值的和，然后将 [p+1, end) 移动到 p 的位置；此时依据从 p 的位置开始看下一个是否可以合并（也就是连续合并的情况）

有几点需要注意：

- 这里将`scanlen`限制为5，也就是说最多合并5个opcode，这是针对一个 VAL opcode 分裂为 3 个 VAL opcode 的情况（即上一张图片中的 case1）
- 连续合并的情况，只在 VAL opcode 的数值更新的情况下发生（即上一张图片中的 case2 和 case3）

### dense 编码

#### 存储格式

dense 编码即使用完整的 16384 个 register。

但是我们存取数据的最小单元是 byte，即 8bit，而每个 register 只有 6bit，那么这 6bit 在 byte 中是怎样的摆放方式呢？

![redis-registers](images/inpost/redis/hll/redis-registers.png)

那怎么取出来呢：

```c
#define HLL_DENSE_GET_REGISTER(target,p,regnum) do {            \
    uint8_t *_p = (uint8_t*) p;                                 \
    unsigned long _byte = regnum*HLL_BITS/8;                    \
    unsigned long _fb = regnum*HLL_BITS&7;                      \
    unsigned long _fb8 = 8 - _fb;                               \
    unsigned long b0 = _p[_byte];                               \
    unsigned long b1 = _p[_byte+1];                             \
    target = ((b0 >> _fb) | (b1 << _fb8)) & HLL_REGISTER_MAX;   \
} while(0)
```

- 首先计算出该 register 所在的第一个字节`_byte`
- 然后计算出该 register 第一个 bit 在该字节中的位置
- 再计算出该 register 在第下一个字节所在的

用图总结大概就是这样：

![redis-register-get](images/inpost/redis/hll/redis-register-get.png)

写入的过程比读取更加麻烦一些：

```c
#define HLL_DENSE_SET_REGISTER(p,regnum,val) do {               \
    uint8_t *_p = (uint8_t*) p;                                 \
    unsigned long _byte = regnum*HLL_BITS/8;                    \
    unsigned long _fb = regnum*HLL_BITS&7;                      \
    unsigned long _fb8 = 8 - _fb;                               \
    unsigned long _v = val;                                     \
    _p[_byte] &= ~(HLL_REGISTER_MAX << _fb);                    \
    _p[_byte] |= _v << _fb;                                     \
    _p[_byte+1] &= ~(HLL_REGISTER_MAX >> _fb8);                 \
    _p[_byte+1] |= _v >> _fb8;                                  \
} while(0)
```

- 首先计算出
- 然后
- 最后

dense 编码下的数据结构各个部分都是确定了的，不存在 sparse 编码中的诸如 opcode 分裂的情况，所以其存取还是比较简单的（当然前提是理解了其存储格式），就不再赘述

## 如何估算

前面提到，HyperLogLog 和 LogLog 的不同之处在于 HLL 使用的是调和平均数，而不是平均数：
$$
H_n = \frac{n}{\frac{1}{x_1} + \frac{1}{x_2} + \frac{1}{x_3} + \dots + \frac{1}{x_n}} = \frac{n}{\sum\limits_{i=1}^{n}{\frac{1}{x_i}}}
$$
HLL 结合了调和平均数的公式：
$$
H_{hll} = const * m * \frac{m}{\sum\limits_{j=1}^{m}\frac{1}{2^{R_j}}}
$$
其中：

- const：用于修正结果的一个常数
- m 表示桶的个数，默认为 16384
- $R_j$ 表示每个桶的估计值
- 

```c
uint64_t hllCount(struct hllhdr *hdr, int *invalid) {
    double m = HLL_REGISTERS;
    double E;
    int j;

    int reghisto[64] = {0};

    if (hdr->encoding == HLL_DENSE) {
        hllDenseRegHisto(hdr->registers,reghisto);
    } else (hdr->encoding == HLL_SPARSE) {
        hllSparseRegHisto(hdr->registers,
                         sdslen((sds)hdr)-HLL_HDR_SIZE,invalid,reghisto);
    } else if (hdr->encoding == HLL_RAW) {
        hllRawRegHisto(hdr->registers,reghisto);
    } else {
        serverPanic("Unknown HyperLogLog encoding in hllCount()");
    }

    double z = m * hllTau((m-registero[HLLL_Q+1])/(double)m)
    for (j = HLL_Q; j >= 1; --j) {
        z += reghisto[j];
        z *= 0.5;
    }
    z += m * hllSigma(reghisto[0]/(double)m)
    E = hllround(HLL_ALPHA_INF*m*m/z)

    return (uint64_t)E;
}
```

1. 首先是以直方图的形式统计值为1、2、3、...、64 的桶各有多少个：
2. 然后是求 z，这个数字其实就是上面公式中的求和部分
3. 最后求 E 的过程，其形式和上面的公式是完全相同的，公式中的 const，就是这里的`HLL_ALPHA_INF`，其值为$\frac{0.5}{ln_2}$

其估算方法来自[这篇论文](https://arxiv.org/pdf/1702.01284.pdf)，由于篇幅太长我数学也太差，就暂时不仔细阅读了，有兴趣的同学可以看看。

## 命令的实现

### `PFCOUNT`命令

前面提到 HLL 一共有三种编码方式，其中 Raw 编码并没有暴露出去，而是只在`PFCOUNT`命令实现函数中用了。所谓 raw 编码，顾名思义就是原始编码，也就是一个字节表示一个 register，当使用`PFCOUNT`命令统计多个 HLL 对象时，redis 会创建一个 raw 编码的临时 hll object，用于将所有需要统计的 hll object 的 register 都统计到这个临时变量中，而后再计算得到估计值。

```c
void pfcountCommand(client *c) {
    robj *o;
    struct hllhdr *hdr;
    uint64_t card;

    if (c->argc > 2) {
        uint8_t max[HLL_HDR_SIZE+HLL_REGISTERS], *registers;
        int j;

        memset(max,0,sizeof(max));
        hdr = (struct hllhdr*) max;
        hdr->encoding = HLL_RAW;
        registers = max + HLL_HDR_SIZE;
        for (j = 1; j < c->argc; j++) {
            robj *o = lookupKeyRead(c->db, c->argv[i]);
            // set max[i] = MAX(max[i],hll[i])
            hllMerge(registers,o);
        }

        addReplyLongLong(c,hllCount(hdr,NULL));
        return;
    }

    o = lookupKeyWrite(c->db, c->argv[1]); 
    hdr = o->ptr;
    if (HLL_VALID_CACHE(hdr)) {
        card = (uint64_t)hdr->card[0];
        card |= (uint64_t)hdr->card[1] << 8;
        ...
    } else {
    	int invalid = 0;
        card = hllCount(hdr,&invalid);
        hdr->card[0] = card & 0xff;
        hdr->card[0] = (card >> 16) & 0xff
        ...
        signalModifiedKey(c,c->db,c->argv[1]);
        server.dirty++;
    }
    addReplayLongLong(c,card);
}
```

首先是统计多个 hll object 的情况，这里并不是单个单个统计而后求和。而是将其转换为单个 hll object 再求和。具体来说，就是使用一个 raw 编码临时 hll object，和每个待统计的 hll object 进行 merge 操作，最终该临时 hll object 中的每个 register 都是所有 hll object 中最大的那个。

而对于统计单个 hll object 的情况，首先检查其 header 中的缓存是否有效，如果仍有效就直接使用，否则需要重新计算估算值并更新缓存（这也是为什么使用`lookupKeyWrite`而不是`lookupKeyRead`的原因）。

## 一些思考

### 编码与数学

### 优化

## 参考

[Redis new data structure: the HyperLogLog](http://antirez.com/news/75)

[HyperLogLog 算法的原理讲解以及 Redis 是如何应用它的](https://juejin.im/post/6844903785744056333)

[Reids(4)——神奇的HyperLoglog解决统计问题](https://www.cnblogs.com/wmyskxz/p/12396393.html)
