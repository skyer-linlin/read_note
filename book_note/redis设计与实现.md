# redis 设计与实现

编辑器和 terminal 跳转：
跳转到编辑器：ctrl+cmd+1
跳转到 terminal：alt+f12

## 第二章 简单动态字符串

字符串对象：

- key：c 语言的基本字符串
- value：SDS（redis 定义的抽象类型，简单动态字符串）

列表：

- key: 一个 SDS
- value：多个 SDS

```c
// SDS 定义
struct sdshdr{
    // 记录数组中已使用字节的数量
    // 等于SDS所保存字符串的长度
    int len;

    // 记录buf数组中未使用的字节的数量
    int free;

    // 字节数组，用来保存字符串
    char buf[];
}
```

### 2.1 SDS 定义

SDS 末尾有“\0"的空字符串，不占用长度，好处是可以使用 c 语言字符串函数库的函数。

相比 c 语言原生字符串的优点：

- 获取字符串长度时间复杂度为 O（1），STRLEN 命令
- 避免缓冲区溢出，操作前会自动检查空间是否足够使用
- 减少修改字符串时带来的内存重分配次数
  - c 语言中，拼接字符串，需要扩展底层数组，否则会缓冲区溢出
  - 截断字符串，需要释放不再使用的空间，否则会内存泄漏
  - redis 存储的数据，可能被频繁修改，如果使用内存重分配，涉及到系统调用，耗时较多，不适合
  - SDS 中，可以有未使用空间，由 free 属性记录
  - 空间预分配：
    - 修改 SDS 后，如果小于 1MB，例如长度为 n，则再分配 n 的空余空间，实际长度为 2n+1
    - 如果大于 1MB，则另外多分配 1MB 空闲空间
  - 惰性空间释放：
    - 当 SDS 有因为操作空闲空间的时候，并不立即释放
- 二进制安全：c 语言使用空字符串（\0）来判断字符串结束，所以不能用来存二进制数据；SDS 可以使用 len 属性来确定字符串的开始和结束，保证读取和写入的数据一致
- 兼容部分 c 语言函数，可以使用 c 语言的字符串头文件的函数实现，免去自己重复实现相关方法

## 第三章 链表

### 链表实现

```c
// 链表节点实现
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

// 链表实现
typedef struct list {
    listNode *head;
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup) (void *ptr);
    // 节点值释放函数
    void (*free) (void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr,void *key);
} list;
```

特点：

- 无环：表头节点的 prev 和表尾节点的 next 指针都指向 NULL，对表的访问以 NULL 为终点
- 获取表头和表尾的复杂度为 1
- 获取某节点的前置后继节点复杂度为 1
- 获取链表长度的复杂度为 1
- 多态：用`void*`保存节点值，可以保存各种不同类型的值

## 第四章 字典

### 4.1 字典的实现

#### 4.1.1 哈希表

```c
typedef struct dictht{
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

#### 4.1.2 哈希表节点

```c
typedef struct dictEntry{
    void *key;

    // 值
    union{
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
}
```

#### 4.1.3 字典

```c
typedef struct dict {
    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privadata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 如果目前没有在进行rehash，那么它的值为-1
    int trehashidx;
} dict;
```

redis 计算 key 的哈希值使用 MurmurHash 算法，对于有规律的输入依然有很好的随机性。

### 4.3 解决键冲突

使用链地址法解决，因为 dictEntry 没有指向链表尾部的指针，所以为了速度考虑，总是将新节点添加在链表头部（如果要增加到尾部，则必须要遍历一遍链表）。

### 4.4 rehash

操作过程：

1. 为 ht[1]哈希表分配空间，扩展则其大小为 ht[0].used\*2 向上取 2 的幂，收缩则为 ht[0].used 向上取 2 的幂
2. 将 ht[0]的所有键值对重新计算哈希值放置到 ht[1]的相应位置上
3. h[0]此时变为空表，释放 ht[0]，将 ht[1]设置为 ht[0]，并在 ht[1]新创建一个空白哈希表

收缩和扩展条件（满足之一即可）：

- 服务器没有在执行 BGSAVE 或者 BGREWRITEAOF 命令，并且哈希表负载因子大于 1
- 服务器目前正在执行 BGSAVE 或者 BGREWRITEAOF 命令，并且哈希表负载银子大于 5
- 当哈希表负载因子小于 0.1，执行收缩操作

### 4.5 渐进式 rehash

如果哈希表中键值对过多，一次性从 ht[0]转移到 ht[1]耗费时间过长，可能导致 redis 在一段时间内不能提供服务

过程：

1. 将 dict 中的 rehashidx 属性从-1 改为 0，表示 rehash 过程开始
2. 开始将 ht[0][rehashidx]的数据从 ht[0]移动到 ht[1]，rehashidx 值加一，继续这一步
3. 当所有键值对都移动完成后，再次将 rehashidx 的值修改为-1

rehash 期间所有的 urd 操作都会在 ht[0]和 ht[1]上同时进行，新增操作则只在 ht[1]上进行。
