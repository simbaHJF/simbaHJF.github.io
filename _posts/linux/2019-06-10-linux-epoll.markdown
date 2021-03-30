---
layout:     post
title:      "LINUX系统epoll函数原理"
date:       2019-06-10 23:00:00 +0800
author:     "simba"
header-img: "img/post-bg-miui6.jpg"
tags:
    - linux

---


在介绍epoll之前,先来说一下select的不足之处,和epoll的改进:
*  一个进程内,select能打开的fd是有限制的,由宏FD_SETSIZE设置,默认值是1024.在某些时候,这个数值是远远不够用的.解决办法有两种,一是修改宏然后重新编译内核,但与此同时会引起网络效率的下降;二是使用多进程来解决,但是创建多个进程是有代价的,而且进程间数据同步没有多线程间方便.
而epoll没有这个限制,它所支持的最大FD上限远远大于1024,在1GB内存的机器上是10万左右(具体数目可以cat /proc/sys/fs/file-max查看).
*  select函数每次都是当监听的套接组有事件产生时就会返回,但却不能将有事件产生的套接字筛选出来,而是改变其在套接组的标志量,所以每次监听到事件,都需要将套接组整个遍历一遍.时间复杂度是O(n).当FD数目增加时,效率会线性下降. 
而epoll,每次会将监听套结字中产生事件的套接字加到一列表中,然后我们可以直接对此列表进行操作,而没有产生事件的套接字会被过滤掉,极大的提高了IO效率.这一点尤其在套接字监听数量巨大而活跃数量很少的时候很明显.


####  epoll的相关系统调用
epoll有epoll_create,  epoll_ctl,  epoll_wait  3个系统调用.
<br>
<br>
a.  int epoll_create(int size);
创建一个epoll的句柄.自从linux2.6.8之后,size参数是被忽略的.需要注意的是,当创建好epoll句柄后,它就是会占用一个fd值,在linux下如果查看/proc/进程id/fd/,是能够看到这个fd的,所以在使用完epoll后,必须调用close()关闭,否则可能导致fd被耗尽.
<br>
<br>
b.  int epoll_ctl(int epfd, int op, int fd, struct epoll_event \*event);
epoll的事件注册函数,它不同于select()是在监听事件时告诉内核要监听什么类型的事件,而是在这里先注册要监听的事件类型.<br>
&emsp;&emsp;第一个参数是epoll_create()的返回值.<br>
&emsp;&emsp;第二个参数表示动作,用三个宏来表示:
*  EPOLL_CTL_ADD:注册新的fd到epfd中;
*  EPOLL_CTL_MOD:修改已经注册的fd的监听事件;
*  EPOLL_CTL_DEL:从epfd中删除一个fd;

&emsp;&emsp;第三个参数是需要监听的fd.<br>
&emsp;&emsp;第四个参数是告诉内核需要监听什么事,struct epoll_event结构如下:
```
//保存触发事件的某个文件描述符相关的数据（与具体使用方式有关）
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;
 //感兴趣的事件和被触发的事件
struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```
&emsp;&emsp;events可以是以下几个宏的集合：
* EPOLLIN:表示对应的文件描述符可以读(包括对端SOCKET正常关闭);
* EPOLLOUT:表示对应的文件描述符可以写;
* EPOLLPRI:表示对应的文件描述符有紧急的数据可读(这里应该表示有带外数据到来);
* EPOLLERR:表示对应的文件描述符发生错误;
* EPOLLHUP:表示对应的文件描述符被挂断;
* EPOLLET:将EPOLL设为边缘触发(Edge Triggered)模式,这是相对于水平触发(Level Triggered)来说的.
* EPOLLONESHOT：只监听一次事件,当监听完这次事件之后,如果还需要继续监听这个socket的话,需要再次把这个socket加入到EPOLL队列里.


c.  int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
收集在epoll监控的事件中已经发送的事件.参数events是分配好的epoll_event结构体数组,epoll将会把发生的事件赋值到events数组中(events不可以是空指针,内核只负责把数据复制到这个events数组中,不会去帮助我们在用户态中分配内存).maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll_create()时的size,参数timeout是超时时间(毫秒,0会立即返回,-1将不确定,也有说法说是永久阻塞).如果函数调用成功,返回对应I/O上已准备好的文件描述符数目,如返回0表示已超时.


####  LT和ET
LT(level triggered)是epoll缺省的工作方式,并且同时支持block和no-block socket.在这种做法中,内核告诉你一个文件描述符是否就绪了,然后你可以对这个就绪的fd进行IO操作.如果你不作任何操作,内核还是会继续通知你的,所以,这种模式编程出错误可能性要小一点.传统的select/poll都是这种模型的代表．
ET (edge-triggered)是高速工作方式,只支持no-block socket,它效率要比LT更高.ET与LT的区别在于,当一个新的事件到来时,ET模式下当然可以从epoll_wait调用中获取到这个事件,可是如果这次没有把这个事件对应的套接字缓冲区处理完,在这个套接字中没有新的事件再次到来时,在ET模式下是无法再次从epoll_wait调用中获取这个事件的.而LT模式正好相反,只要一个事件对应的套接字缓冲区还有数据,就总能从epoll_wait中获取这个事件.
因此,LT模式下开发基于epoll的应用要简单些,不太容易出错.而在ET模式下事件发生时,如果没有彻底地将缓冲区数据处理完,则会导致缓冲区中的用户请求得不到响应.
Nginx默认采用ET模式来使用epoll.


####  epoll工作原理
epoll只告知那些就绪的文件描述符,而且当我们调用epoll_wait()获得就绪文件描述符时,返回的不是实际的描述符,而是一个代表就绪描述符数量的值,只需要去epoll指定的一个数组中依次取得相应数量的文件描述符即可.<br>

另一个本质的改进在于epoll采用基于事件的就绪通知方式.在select/poll中,进程只有在调用一定的方法后(比如调用select方法),内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符,一旦基于某个文件描述符就绪时,内核会采用类似callback的回调机制,迅速激活这个文件描述符,当进程调用epoll_wait()时便得到通知.

当一个进程调用epoll_create方法时,Linux内核会创建一个eventpoll结构体,这个结构体中有两个成员与epoll的使用方式密切相关:
```
 /*
  * This structure is stored inside the "private_data" member of the file  
  * structure and represents the main data structure for the eventpoll  
  * interface. 
  */   
struct eventpoll {  
  
	/* Protect the access to this structure */    
	spinlock_t lock;  
	   
	  
	/*  
	 * This mutex is used to ensure that files are not removed  
	 * while epoll is using them. This is held during the event  
	 * collection loop, the file cleanup path, the epoll file exit 
	 * code and the ctl operations. 
	 */    
	struct mutex mtx;  
	
	  
	/* Wait queue used by sys_epoll_wait() */    
	wait_queue_head_t wq;    
	  
	  
	/* Wait queue used by file->poll() */    
	wait_queue_head_t poll_wait;    
	  
	  
	/* List of ready file descriptors */    
	struct list_head rdllist;  
	  
	  
	/* RB tree root used to store monitored fd structs */    
	struct rb_root rbr;//红黑树根节点，这棵树存储着所有添加到epoll中的事件，也就是这个epoll监控的fd  
	 
	/* 
	 * This is a single linked list that chains all the "struct epitem" that 
	 * happened while transferring ready events to userspace w/out 
	 * holding ->lock. 
	 */  
	struct epitem *ovflist;  
	 
	/* wakeup_source used when ep_scan_ready_list is running */  
	struct wakeup_source *ws;  
	
	/* The user that created the eventpoll descriptor */  
	struct user_struct *user;  
	 
	struct file *file;  
	
	/* used to optimize loop detection check */  
	int visited;  

	struct list_head visited_list_link;//双向链表中保存着将要通过epoll_wait返回给用户的、满足条件的事件
};
```

每一个epoll对象都有一个独立的eventpoll结构体,这个结构体会在内核空间中创造独立的内存,用于存储使用epoll_ctl方法向epoll对象中添加进来的事件.这样,重复的事件就可以通过红黑树而高效的识别出来.在epoll中,对于每一个事件都会建立一个epitem结构体:
```
/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the "rbr" RB tree.
 * Avoid increasing the size of this struct, there can be many thousands
 * of these on a server and we do not want this to take another cache line.
 */
struct epitem {
        /* RB tree node used to link this structure to the eventpoll RB tree */
        struct rb_node rbn;

        /* List header used to link this structure to the eventpoll ready list */
        struct list_head rdllink;

        /*
         * Works together "struct eventpoll"->ovflist in keeping the
         * single linked chain of items.
         */
        struct epitem *next;

        /* The file descriptor information this item refers to */
        struct epoll_filefd ffd;

        /* Number of active wait queue attached to poll operations */
        int nwait;

        /* List containing poll wait queues */
        struct list_head pwqlist;

        /* The "container" of this item */
        struct eventpoll *ep;

        /* List header used to link this item to the "struct file" items list */
        struct list_head fllink;

        /* wakeup_source used when EPOLLWAKEUP is set */
        struct wakeup_source __rcu *ws;

        /* The structure that describe the interested events and the source fd */
        struct epoll_event event;
};
```
epoll维护了一个双链表,用户存储发生的事件.当epoll_wait调用时,仅仅观察这个list链表里有没有数据即可.有数据就返回,没有数据就sleep,等到timeout时间到后即使链表没数据也返回.所以,epoll_wait非常高效.
通常情况下即使我们要监控百万计的句柄,大多一次也只返回很少量的准备就绪句柄而已,所以,epoll_wait仅需要从内核态copy少量的句柄到用户态而已,如何能不高效?
这个准备就绪list链表是怎么维护的呢?当我们执行epoll_ctl时,除了把socket放到epoll文件系统里file对象对应的红黑树上之外,还会给内核中断处理程序注册一个回调函数,告诉内核,如果这个句柄的中断到了,就把它放到准备就绪list链表里.所以,当一个socket上有数据到了,内核在把网卡上的数据copy到内核中后就把socket插入到准备就绪链表里了.
如此,一颗红黑树,一张准备就绪句柄链表,少量的内核cache,就帮我们解决了大并发下的socket处理问题.执行epoll_create时,创建了红黑树和就绪链表,执行epoll_ctl时,如果增加socket句柄,则检查在红黑树中是否存在,存在立即返回,不存在则添加到树干上,然后向内核注册回调函数,用于当中断事件来临时向准备就绪链表中插入数据.执行epoll_wait时立刻返回准备就绪链表里的数据即可.


<br>
<br>

参考文章:<br>
https://blog.csdn.net/xiajun07061225/article/details/9250579
https://blog.csdn.net/to_be_better/article/details/47349573
