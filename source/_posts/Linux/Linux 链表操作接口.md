title: Linux 链表操作接口
date: 2018-11-9 21:00:00
tags: Linux

---

[TOC]

前提是假设大家有链表的基础
https://www.ibm.com/developerworks/cn/linux/kernel/l-chain/index.html

## 声明和初始化
Linux 只定义了链表的节点, 没有专门定义链表头.
链表结构是这样建立的
### 声明时初始化链表 LIST_HEAD_INIT
```
#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)
```
`LIST_HEAD_INIT(name)`声明一个名为 name 的链表头时，它的next、prev指针都初始化为指向自己 , 这样，我们就有了一个空链表.
(Linux用头指针的next是否指向自己来判断链表是否为空)
```
static inline int list_empty(const struct list_head *head)
{
        return head->next == head;
}
```
### 运行时初始化链表 INIT_LIST_HEAD
```
#define INIT_LIST_HEAD(ptr) do { \
    (ptr)->next = (ptr); (ptr)->prev = (ptr); \
} while (0)
```

## 插入 删除 搬移 合并
### 插入

对链表的插入操作有两种：在表头插入和在表尾插入。Linux为此提供了两个接口：
```
static inline void list_add(struct list_head *new, struct list_head *head);
static inline void list_add_tail(struct list_head *new, struct list_head *head);
```
因为Linux链表是循环表，且表头的next、prev分别指向链表中的第一个和最末一个节点，所以，list_add和list_add_tail的区别并不大，实际上，Linux分别用
```
__list_add(new, head, head->next);
__list_add(new, head->prev, head);
```
来实现两个接口，可见，在表头插入是插入在head之后，而在表尾插入是插入在head->prev之后。
假设有一个新nf_sockopt_ops结构变量new_sockopt需要添加到nf_sockopts链表头，我们应当这样操作：
```
list_add(&new_sockopt.list, &nf_sockopts);
```
从这里我们看出，nf_sockopts链表中记录的并不是new_sockopt的地址，而是其中的list元素的地址。
如何通过链表访问到new_sockopt呢？下面会有详细介绍。

### 删除
```
static inline void list_del(struct list_head *entry);
```
当我们需要删除nf_sockopts链表中添加的new_sockopt项时，我们这么操作：
```
list_del(&new_sockopt.list);
```
被剔除下来的new_sockopt.list，prev、next指针分别被设为LIST_POSITION2和LIST_POSITION1两个特殊值，
这样设置是为了保证不在链表中的节点项不可访问--对LIST_POSITION1和LIST_POSITION2的访问都将引起页故障。
与之相对应，list_del_init()函数将节点从链表中解下来之后，调用LIST_INIT_HEAD()将节点置为空链状态。

### 搬移
Linux提供了将原本属于一个链表的节点移动到另一个链表的操作，并根据插入到新链表的位置分为两类：
```
static inline void list_move(struct list_head *list, struct list_head *head);
static inline void list_move_tail(struct list_head *list, struct list_head *head);
```
例如list_move(&new_sockopt.list,&nf_sockopts)会把new_sockopt从它所在的链表上删除，并将其再链入nf_sockopts的表头。

### 合并
除了针对节点的插入、删除操作，Linux链表还提供了整个链表的插入功能：
```
static inline void list_splice(struct list_head *list, struct list_head *head);
```
假设当前有两个链表，表头分别是list1和list2（都是struct list_head变量），当调用list_splice(&list1,&list2)时，只要list1非空，list1链表的内容将被挂接在list2链表上，位于list2和list2.next（原list2表的第一个节点）之间。新list2链表将以原list1表的第一个节点为首节点，而尾节点不变。如图（虚箭头为next指针）：

图4 链表合并list_splice(&list1,&list2)
![链表合并list_splice](https://ws1.sinaimg.cn/large/ba061518gy1fx1lqrf07xg20fc04pmwy.gif)
当list1被挂接到list2之后，作为原表头指针的list1的next、prev仍然指向原来的节点，为了避免引起混乱，Linux提供了一个list_splice_init()函数：
```
static inline void list_splice_init(struct list_head *list, struct list_head *head);
```
该函数在将list合并到head链表的基础上，调用INIT_LIST_HEAD(list)将list设置为空链。

## 遍历
### 由链表节点到数据项变量
我们知道，Linux链表中仅保存了数据项结构中list_head成员变量的地址，那么我们如何通过这个list_head成员访问到作为它的所有者的节点数据呢？
Linux为此提供了一个list_entry(ptr,type,member)宏，
ptr是指向该数据中list_head成员的指针，也就是存储在链表中的地址值，
type是数据项的类型，
member则是数据项类型定义中list_head成员的变量名，
例如，我们要访问nf_sockopts链表中首个nf_sockopt_ops变量，则如此调用：
```
list_entry(nf_sockopts->next, struct nf_sockopt_ops, list);
```
这里"list"正是nf_sockopt_ops结构中定义的用于链表操作的节点成员变量名。

list_entry的使用相当简单，相比之下，它的实现则有一些难懂：
```
#define list_entry(ptr, type, member) container_of(ptr, type, member)
```
container_of宏定义在[include/linux/kernel.h]中：
```
#define container_of(ptr, type, member) ({          \
        const typeof( ((type *)0)->member ) *__mptr = (ptr); \
        (type *)( (char *)__mptr - offsetof(type,member) );})
```
offsetof宏定义在[include/linux/stddef.h]中：
```
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```
size_t最终定义为unsigned int（i386）。

这里`container_of`使用的是一个利用编译器技术的小技巧，即先求得结构成员在与结构中的偏移量，然后根据成员变量的地址反过来得出属主结构变量的地址。

> container_of()和offsetof()并不仅用于链表操作，
> 这里最有趣的地方是`((type *)0)->member`，它将0地址强制"转换"为type结构的指针，再访问到type结构中的member成员。
> 在container_of宏中，它用来给typeof()提供参数（typeof()是gcc的扩展，和sizeof()类似），以获得member成员的数据类型；
> 在offsetof()中，这个member成员的地址实际上就是type数据结构中member成员相对于结构变量的偏移量。

如果这么说还不好理解的话，不妨看看下面这张图：
图5 offsetof()宏的原理
![](https://ws1.sinaimg.cn/large/ba061518gy1fx1ls58gc9g20fc05agle.gif)
对于给定一个结构，offsetof(type,member)是一个常量，list_entry()正是利用这个不变的偏移量来求得链表数据项的变量地址。

### 遍历宏 list_for_each_entry()
list_for_each_entry = list_for_each() + list_entry()

在[net/core/netfilter.c]的nf_register_sockopt()函数中有这么一段话：
```
……
struct list_head *i;
……
    list_for_each(i, &nf_sockopts) {
        struct nf_sockopt_ops *ops = (struct nf_sockopt_ops *)i;
        ……
    }
……
```
函数首先定义一个`(struct list_head *)`指针变量i，然后调用list_for_each(i,&nf_sockopts)进行遍历。
在[include/linux/list.h]中，list_for_each()宏是这么定义的：
```
#define list_for_each(pos, head) \
for (pos = (head)->next, prefetch(pos->next); pos != (head); \
        pos = pos->next, prefetch(pos->next))
```
它实际上是一个for循环，利用传入的pos作为循环变量，从表头head开始，逐项向后（next方向）移动pos，直至又回到head（prefetch()可以不考虑，用于预取以提高遍历速度）。

那么在nf_register_sockopt()中实际上就是遍历nf_sockopts链表。为什么能直接将获得的list_head成员变量地址当成struct nf_sockopt_ops数据项变量的地址呢？我们注意到在struct nf_sockopt_ops结构中，list是其中的第一项成员，因此，它的地址也就是结构变量的地址。更规范的获得数据变量地址的用法应该是：
```
struct nf_sockopt_ops *ops = list_entry(i, struct nf_sockopt_ops, list);
```
大多数情况下，遍历链表的时候都需要获得链表节点数据项，也就是说list_for_each()和list_entry()总是同时使用。

对此Linux给出了一个`list_for_each_entry()`宏：
```
#define list_for_each_entry(pos, head, member)      ……
```
与list_for_each()不同，这里的pos是数据项结构指针类型，而不是`(struct list_head *)`。
nf_register_sockopt()函数可以利用这个宏而设计得更简单：
```
……
struct nf_sockopt_ops *ops;
list_for_each_entry(ops,&nf_sockopts,list){
    ……
}
……
```
某些应用需要反向遍历链表，Linux提供了list_for_each_prev()和list_for_each_entry_reverse()来完成这一操作，使用方法和上面介绍的list_for_each()、list_for_each_entry()完全相同。

如果遍历不是从链表头开始，而是从已知的某个节点pos开始，则可以使用list_for_each_entry_continue(pos,head,member)。有时还会出现这种需求，即经过一系列计算后，如果pos有值，则从pos开始遍历，如果没有，则从链表头开始，为此，Linux专门提供了一个list_prepare_entry(pos,head,member)宏，将它的返回值作为list_for_each_entry_continue()的pos参数，就可以满足这一要求。