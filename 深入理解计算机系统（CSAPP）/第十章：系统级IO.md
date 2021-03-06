## I/O

所有的I/O设备都被模型化为文件。

包括：磁盘，网络，终端。

磁盘是磁盘控制器连在主板上的，网络是通过网络适配器连在主板上的，终端是什么？这么想，想不通，想不通就不想，当作概念好了。

* 打开文件——一个应用程序通过要求内核打开相应的文件。——文件操作总是内核做的。应用只是告诉内核做什么。——文件打开后，应用程序记录描述符，内核记录所有和文件相关的信息。
每个进程开始的时候都有3个打开的文件：0，1，2，入，出，错出。
* 改变当前的文件位置——对于每个打开的文件，内核记录其所有相关的信息，其中一个是文件位置，初始为0。——从文件开头起始的字节偏移量。
* 读文件——一个读操作就是从文件拷贝n个字节到存储器，从当前文件位置k开始，然后将k增加到k+n。
给定的一个大小为m字节的文件，当k>=m时，执行读操作会触发一个称为EOF的条件，应用程序能检测到这个条件。——EOF是一个条件，不是一个字节，不是一个位，在文件的结尾处什么都没有，也没有EOF符号，EOF是一个条件。
* 写操作——从存储器拷贝n>0个字节到一个文件，从当前文件k开始，然后更新k。
* 关闭文件——应用程序通知内核，让内核去关闭文件。内核会释放其记录的关于文件的所有。

##  打开和关闭文件

进程通过open函数来打开一个已存在的文件或者创建一个新文件。

int open(char *filename, int flags, int mode);

flag有：O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC, O_APPEND。

每个进程都有一个 umask，这是上下文的一部分。

open函数创建一个新文件时，文件的访问权限位被设置成mode & ~umask。

## 读和写文件

应用程序通过分别调用read和write函数来执行输入和输出。

ssize_t read(int fd, void *buf, size_t n);
ssize_t write(int fd, const void *buf, size_t n);
read函数调用返回有三种情况：

-1——错误。
0——调用的时候，触发了k>=m的EOF条件，应该这么说，调用的时候，当前位置k已经在文件结尾了。
实际传送的字节数——这里又有两种情况，一种是n，也就是实际传送了n个；另一种是小于n的，也就是实际没有传送到n那么多，导致这种情况的原因可能就是从当前位置到文件尾部之间的字节数没有n那么多了（在这种情况下，下一次再read，直接就返回0了）。
如果打开文件是与终端相关联的，那么每个read函数将一次传送一个文本行，返回的不足值对于文本行的大小。

## 用RIO包健壮地读写

Robust I/O，RIO提供了两类不同的函数：

无缓冲的输入输出函数——这些函数直接在存储器和文件之间传送数据，没有应用级缓冲。当文件是网络时，尤其有用。
带缓冲的输入函数——文件的内容缓存在应用级缓冲区内。
无缓冲的函数：

ssize_t rio_readn(int fd, void  *usrbuf, size_t n);
ssize_t rio_writen(int fd, void *usrbuf, size_t n);
rio_readn比read好的地方在于：信号中断会重启。

带缓冲的输入函数（没有输出函数）

结构rio_t。

void rio_readinitb(rio_t *rp, int fd);
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n);
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);

## 读取文件元数据

文件的元数据，就是文件的相关信息。

int stat(const char *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
结构stat中，我们关心的，是st_mode和st_size。st_mode编码了文件访问许可位和文件类型。

Unix识别大量不同的文件类型。

普通文件：文本文件和二进制文件。
目录文件
套接字

## 共享文件

内核用三个相关的数据结构来表示打开的文件：描述符表，文件表，v-node表。

描述符表是每个进程都有的。文件表和v-node表是所有进程共享的。

描述符表的每个条目是由文件描述符索引的，条目值指向文件表中的一项。

文件表的条目包含一个文件的当前位置，引用计数，以及一个指向v-node表中对应表项的指针。

也就是说，如果两个进程的描述符各自有一个条目指向文件表中的一个，那么这两个进程就共享了文件的当前位置这一属性。一个read了，当前位置变了，另一个读的时候，这种变化会体现出来。

v-node表，包含了文件的元数据，也就是，stat结构的内容。

一次open，将在文件表中建立一个新的条目，将在描述符表中建立一个新的描述符指向这个文件表中的条目。

如果已open之后，进程fork了，那么父子进程都拥有相同的描述符内容，指向的也会是文件表中相同的条目，也就是共享了该文件的当前位置。

两次open同一个文件，虽然是同一个文件，但是有两个描述符条目，指向的也是文件表中的两个条目，虽然文件表中的两个条目指向v-node表中的一个条目，但是，文件表两个条目是关键。

## I/O重定向

I/O重定向的一个方式是使用dup2函数。

int dup2(int oldfd, int newfd);

oldfd将覆盖newfd。也就是说newfd将指向和oldfd相同的文件表的某一项，oldfd保持不变。再简单点——后者变成和前者一样的了。

## 标准I/O

ANSI C定义了一组高级输入输出函数，称为标准I/O库。

标准I/O库将一个打开的文件模型化为一个流。

对于程序员而言，一个流就是一个指向FILE类型的结构的指针。

类型为FILE的流是对文件描述符和流缓冲区的抽象。——这里，和前面的rio_t有点类似了，FILE将文件描述符和缓冲区合在了一起，而rio_t将文件描述符和缓冲区合在了一起，还是很类似的。