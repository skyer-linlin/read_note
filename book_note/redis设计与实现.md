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
