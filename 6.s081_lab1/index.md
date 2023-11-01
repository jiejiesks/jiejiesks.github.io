# 6.s081_lab1


## lab1

### sleep

- 思路

直接将argv[1]赋值给sleep系统调用即可

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    char *sleeptime_char;
    int sleeptime;
    if (argc <= 1)
    {
        printf("sleep need one parm");
        exit(1);
    }
    if (argc > 2)
    {
        printf("too many parm");
        exit(1);
        /* code */
    }

    sleeptime_char = argv[1];
    sleeptime = atoi(sleeptime_char);
    sleep(sleeptime);
    exit(0);
}
```

### pingpong

在pingpong.c中，有两种思路

- 思路一

只创建一个管道，父子都通过这个管道进行读写，父进程在写入之后必须wait等待子进程读取并在写入之后，在进行读取操作。

```c
#include "kernel/types.h"
#include "user.h"
 
int main(int argc,char* argv[]){
    //创建两个管道，分别实现ping、pong的读写
    int p[2];
    pipe(p);
    char readtext[10];//作为父进程和子进程的读出容器
    //子程序读出
    int pid = fork();
    if(pid==0){
        read(p[0],readtext,10);
        printf("%d: received %s\n",getpid(),readtext);
        write(p[1],"pong",10);
        exit(0);//子进程一定要退出
    }
    //父程序写入
    else{
        write(p[1],"ping",10);
        wait(0);//父进程阻塞，等待子进程读取
        read(p[0],readtext,10);
        printf("%d: received %s\n",getpid(),readtext);
        exit(0);//父进程一定要退出
    }
    return 0;
}
```



- 思路二

需要建立两个管道分别用到作为读写管道。管道一头只作为读，另一头只作为写。当父子进程都需要读写时，创建两个管道，一个父读子写，一个父写子读。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    //0 read 1 write
    int child_fd[2];
    int parent_fd[2];

    pipe(child_fd);
    pipe(parent_fd);

    
    if (fork()==0)
    {   
        char buf[80];
        close(child_fd[1]);
        read(child_fd[0],buf,sizeof(buf));
        close(child_fd[0]);
        printf("%d: received p%sng\n",getpid(),buf);
        close(parent_fd[0]);
        write(parent_fd[1],"o",1);
        close(parent_fd[1]);
        exit(0);
    }
    else 
    {
        
        char buf[80];
        close(child_fd[0]);
        write(child_fd[1],"i",1);
        close(child_fd[1]);

        close(parent_fd[1]);
        read(parent_fd[0],buf,sizeof(buf));
        close(parent_fd[0]);
        printf("%d: received p%sng\n",getpid(),buf);
        exit(0);

    }
    return 0;
    
}
```

其中思路二更好理解，两个管道各司其职

### primes

- 思路

main函数中负责创建一个管道，将2-35全部写入管道中。在child process中负责调用一个创建子进程的函数。

该函数主要做以下几件事：

1. 读取parent process送入的每个数字，将其打印prime i。不用担心main函数的顺序，因为如果main函数没有write数字也没有关闭写描述符，那么child process的read函数会阻塞。
2. 继续读取parent process管道中的数字，如果没有数字代表不需要在创建child process，退出。如果仍有数字，继续创建child process。
3. 通过fork函数创建child process，在当前的process中需要判断该数字是否为素数，为素数则写入子进程的管道中。只有在数字2创建的进程需要一一判断，后续进程皆为素数产生的进程。

Tips:如果有指向管道写端的文件描述符没有关闭，而持有写端的进程也没有向管道内写入数据的时候，那么管道剩余的数据被读取后，再次read会被阻塞，之后有数据可读会再次返回。 如果所有的写端均被关闭，那么再次read会直接返回0，就像读到了文件末尾一样。

如果管道内没有其他prime，则不再创建子进程

注意wait，必须等到所有child process都结束了parent process才能结束

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int isprime(int n)
{
    int j, flag = 1;

    for (j = 2; j <= n / 2; ++j)
    {
        if (n % j == 0)
        {
            flag = 0;
            break;
        }
    }
    return flag;
}

int create_process(int parent_fd[2])
{
    char bytes[4];
    close(parent_fd[1]);
    /*如果有指向管道写端的文件描述符没有关闭，而持有写端的进程也没有向管道内
    写入数据的时候，那么管道剩余的数据被读取后，再次read会被阻塞，之后有数据
    可读会再次返回。
    如果所有的写端均被关闭，那么再次read会直接返回0，就像读到了文件末尾一样。
    */
    read(parent_fd[0], bytes, sizeof(bytes));
    int prime = *(int *)bytes;
    printf("prime %d\n", prime);
    /*如果管道内没有其他prime，则不再创建子进程*/
    if (read(parent_fd[0], bytes, sizeof(bytes)) == 0)
    {
        close(parent_fd[0]);
        exit(0);
    }

    int child_fd[2];
    pipe(child_fd);

    int pid = fork();
    if (pid < 0)
    {
        printf("fork error");
        exit(1);
    }
    else if (pid == 0)
    {
        create_process(child_fd);
        exit(0);
    }
    else
    {
        close(child_fd[0]);
        do
        {
            int selectprime = *(int *)bytes;
            if (isprime(selectprime))
            {
                write(child_fd[1], bytes, sizeof(bytes));
            }
        } while (read(parent_fd[0], bytes, sizeof(bytes)));
        close(parent_fd[0]);
        close(child_fd[1]);
        //必须等到所有子进程都结束了才能exit
        wait((int *)0);
        exit(0);
    }
}
int main(int argc, char *argv[])
{
    int parent_fd[2];
    pipe(parent_fd);
    int count = 35;
    int pid = fork();
    if (pid < 0)
    {
        printf("fork error");
        exit(1);
    }
    else if (pid == 0)
    {
        create_process(parent_fd);
        exit(0);
    }
    else
    {
        close(parent_fd[0]);
        for (int i = 2; i <= count; i++)
        {
            char bytes[4];
            bytes[3] = (i >> 24) & 0xff;
            bytes[2] = (i >> 16) & 0xff;
            bytes[1] = (i >> 8) & 0xff;
            bytes[0] = i & 0xff;
            write(parent_fd[1], bytes, sizeof(bytes));
        }
        close(parent_fd[1]);
        wait((int *)0);
        exit(0);
    }
}

```



### find

- 思路

  find(*path, *filename)获取想要查找的目录路径，和目标文件名

- - 创建缓冲区和一系列变量，将路径存储到缓冲区buf中，指针p指向最后一个/的位置

  - open打开路径path到句柄fd

  - stat将fd的stat读取到st中

  - 循环体while，判断条件为是否成功从句柄fd中读取dirent结构（目录层），使用read()函数将读取到的dirent存储到de中

  - - 忽略de.inum==0的项

    - 拼接de.name到buf末尾，获得fd指向的目录下的一个文件完整路径

    - stat访问完整路径，存入st对st.type进行判断，进入switch结构

    - - 当st.type为T_FILE即文件时

      - - 对比当前文件名（de.name）和目标文件名（targent）是否一致
        - 如果一致，输出buf中的完整目录
        - break

      - 当st.type为T_DIR即目录时

      - - 迭代find(buf,targent)
        - break

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

int find(char *path, char *filename)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;
    int findflag = 0;

    if ((fd = open(path, 0)) < 0)
    {
        fprintf(2, "find: cannot open %s\n", path);
        return -1;
    }

    if (fstat(fd, &st) < 0)
    {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return -1;
    }
    // 将P指针指向path的最后buf+1的位置赋值"/"
    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';
    while (read(fd, &de, sizeof(de)) == sizeof(de))
    {
        if (de.inum == 0)
        {
            continue;
        }

        if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
        {
            continue;
        }
        // 将指定路径目录下的dirent的name赋值给p，由此buf获取完整的路径
        memmove(p, de.name, DIRSIZ);
        *(p + DIRSIZ) = 0;
        // 可以通过完整的文件名获取文件的stat，由此可以获取其type
        if (stat(buf, &st) < 0)
        {
            printf("find: cannot stat %s\n", buf);
            continue;
        }
        switch (st.type)
        {
        case T_FILE:
            if (strcmp(de.name, filename) == 0)
            {
                printf("%s\n", buf);
                findflag = 1;
            }
            break;
        case T_DIR:
            findflag = find(buf, filename);
            break;
        }
    }
    close(fd);
    return findflag;
}
int main(int argc, char *argv[])
{
    int findflag = 0;
    if (argc < 2)
    {
        printf("find need para\n");
        exit(1);
    }
    else if (argc == 2)
    {
        findflag = find(".", argv[1]);
    }
    else if (argc == 3)
    {
        findflag = find(argv[1], argv[2]);
    }
    else
    {
        printf("too many para\n");
        exit(1);
    }

    if (findflag == 0)
    {
        printf("can not find the file\n");
        exit(1);
    }
    exit(0);
}
```



### xargs

- 思路

以find . | xargs grep hello为例

xargs之需要处理｜之后的参数，而｜之前的参数保存在line中，创建一个字符容量为4的字符数组，1:cmd即例子中的grep，2:argv[2]即例子中的hello，3:line即例子中find .的结果，4:0代表结束。通过while持续读一个字符到line字符数组中，直到读到换行符结束。之后在子进程中调用exec系统调用，参数即为cmd和容量为4的字符数组。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/param.h"
#include "user/user.h"

#define MAXLINE 32
int main(int argc, char *argv[])
{
    if (argc < 3)
    {
        printf("xargs need at least three params");
        exit(1);
    }
    char *cmd = argv[1];
    char line[MAXLINE];
    // 管道前产生的结果放入line中
    memset(line, 0, sizeof(line));

    // 读入的每一个char
    int i = 0;
    char ch;
    while (read(0, &ch, sizeof(ch)) != 0)
    {
        if (ch == '\n')
        {
            char *child_argv[4];
            // 1: cmd 2: argv[2] 3:line 4:0表示结束
            child_argv[0] = cmd;
            child_argv[1] = argv[2];
            child_argv[2] = line;
            child_argv[3] = 0;
            if (fork() == 0)
            {
                exec(cmd, child_argv);
            }
            else
            {
                wait((int *)0);
                // parent process need to wait child process
                
            }
            memset(line,0,sizeof(line));
            i=0;
        }
        else
        {
            line[i++] = ch;
        }
    }
    exit(0);
}
```

Tips:字符之间可以用==比较，可以用=赋值

字符串之间不可以用==比较，要用strcmp

字符串比较

```c
int main()
{
    char *str1="hello";
    char str2[]="hello";
    printf("%d\n",str1=="hello");
    printf("%d\n",str2=="hello");
    printf("%d\n",strcmp(str1,"hello"));
    printf("%d\n",strcmp(str2,"hello"));

​	return 0；

}
```

输出结果为1 0 0 0
1.字符串变量比较不能直接用==，但是可以用变量地址和字符串用==比较，如果地址相同，字符串会相等

char *str1 = “hello”;和”hello”的地址是相同的，所以返回结果相等

str2 == “hello”地址不相等。char str2[] = “hello”; 这里str2并不是指针，类型里已经说明它是一个数组，所以这会是另一个内存地址，于是str2与”hello”的地址是不同的。

综上：字符串的比较不能用==


2.字符串比较用strcmp函数

strcmp(str1,”hello”)，strcmp(str2,”hello”)都是成立的
由于”hello”是字符串常量，编译器会进行优化：
所有的”hello”都是相同的，整个程序中只需要有一个”hello”字符串。

然后所有引用”hello”这个字符串的“指针变量”都赋值成相同的地址。

3.字符串赋值不能用=

