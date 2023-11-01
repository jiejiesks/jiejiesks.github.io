# Read_write_struct


# 关于结构体的读写

## user space

- 可以通过fread和rwrite进行结构体的读写。

```c
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream)
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream)


ptr − This is the pointer to a block of memory with a minimum size of size*nmemb bytes.

size − This is the size in bytes of each element to be read.

nmemb − This is the number of elements, each one with a size of size bytes.

stream − This is the pointer to a FILE object that specifies an input stream.
```

代码示例

```c
#include <stdio.h>

/* 定义结构体, 存储一个字符串和年龄 */
struct student
{
    char name[20];
    int age;
};

int main()
{
    // 要写入文件的结构体
    struct student s1 = {"Tom", 18};

    // 打开要写入的文件
    FILE *p = fopen("D:/File/student.dat", "w");
    // 打开失败直接退出
    if(p == NULL)
        return 0;

    // 将结构体写出到文件中
    fwrite(&s1, sizeof (struct student), 1 ,p);
    // 关闭文件
    fclose(p);
    // 读取文件中的结构体
    // 存储读取到的结构体数据
    struct student s2 = {0};

    // 打开文件
    FILE *p2 = fopen("D:/File/student.dat", "r");
    // 如果打开失败, 退出
    if(p2 == NULL)
        return 0;

    // 从文件中读取结构体信息
    fread(&s2, sizeof (struct student), 1 ,p2);
    // 关闭文件
    fclose(p2);

    // 打印数据
    printf("student : name=%s, age=%d\n", s2.name, s2.age);

    return 0;
}
```

## kernel

- kernel中并不像user space中那么简单。

- 打开文件

  filp_open()在kernel中可以打开文件，其原形如下：

  ```c
  strcut file* filp_open(const char* filename, int open_mode, int mode);
  ```

  该函数返回strcut file*结构指针，供后继函数操作使用，该返回值用IS_ERR()来检验其有效性。

  参数说明

  filename： 表明要打开或创建文件的名称(包括路径部分)。在内核中打开的文件时需要注意打开的时机，很容易出现需要打开文件的驱动很早就加载并打开文件，但需要打开的文件所在设备还不有挂载到文件系统中，而导致打开失败。

  open_mode： 文件的打开方式，其取值与标准库中的open相应参数类似，可以取O_CREAT,O_RDWR,O_RDONLY等。

  mode： 创建文件时使用，设置创建文件的读写权限，其它情况可以匆略设为0

- 读写文件

  kernel中文件的读写操作可以使用vfs_read()和vfs_write，在使用这两个函数前需要说明一下get_fs()和 set_fs()这两个函数。

  vfs_read() vfs_write()两函数的原形如下：

  ```c
  ssize_t vfs_read(struct file* filp, char __user* buffer, size_t len, loff_t* pos);
  
  ssize_t vfs_write(struct file* filp, const char __user* buffer, size_t len, loff_t* pos);
  ```

  注意这两个函数的第二个参数buffer，前面都有__user修饰符，这就要求这两个buffer指针都应该指向用空的内存，如果对该参数传递kernel空间的指针，这两个函数都会返回失败-EFAULT。但在Kernel中，我们一般不容易生成用户空间的指针，或者不方便独立使用用户空间内存。要使这两个读写函数使用kernel空间的buffer指针也能正确工作，需要使用set_fs()函数或宏(set_fs()可能是宏定义)，如果为函数，其原形如下：

  ```c
  void set_fs(mm_segment_t fs);
  ```

  该函数的作用是改变kernel对内存地址检查的处理方式，其实该函数的参数fs只有两个取值：USER_DS，KERNEL_DS，分别代表用户空间和内核空间，默认情况下，kernel取值为USER_DS，即对用户空间地址检查并做变换。那么要在这种对内存地址做检查变换的函数中使用内核空间地址，就需要使用set_fs(KERNEL_DS)进行设置。get_fs()一般也可能是宏定义，它的作用是取得当前的设置，这两个函数的一般用法为：

  ```c
  　mm_segment_t old_fs;
  
  　old_fs = get_fs();
  
  　set_fs(KERNEL_DS);
  
  　...... //与内存有关的操作
  
  　set_fs(old_fs);
  ```

  还有一些其它的内核函数也有用__user修饰的参数，在kernel中需要用kernel空间的内存代替时，都可以使用类似办法。

  使用vfs_read()和vfs_write()最后需要注意的一点是最后的参数loff_t * pos，pos所指向的值要初始化，表明从文件的什么地方开始读写。

  在新版的kernel中，通过kernel_read和kernel_write封装上述烦琐的过程。

  ```c
  ssize_t kernel_read(struct file *file, void *buf, size_t count, loff_t *pos)
  {
      mm_segment_t old_fs;
      ssize_t result;
   
      old_fs = get_fs();
      set_fs(KERNEL_DS);
      /* The cast to a user pointer is valid due to the set_fs() */
      result = vfs_read(file, (void __user *)buf, count, pos);
      set_fs(old_fs);
      return result;
  }
  ```

- 在将结构体写入文件时，由于指针所指向的空间已经被内容填充，直接调用函数即可。

  ```c
  writefile = filp_open("~/Desktop/file_and_path.txt", O_RDWR | O_APPEND | O_CREAT, 0);
  if (IS_ERR(writefile))
      {
          printk(KERN_ERR "open dir %s error.\n", path);
          return -1;
      }
  kernel_write(writefile, file, sizeof(struct file), &wpos);
  ```

  

- 在读入文件时，定义读入的指针时需要通过kmalloc分配空间，否则会出现空指针的BUG。

  ```c
  struct file *readfile = NULL;
  struct file *testfile = kmalloc(sizeof(struct file), GFP_KERNEL);
  loff_t rpos = 0;
  readfile = filp_open("/home/zxj/Desktop/file_and_path.txt", O_RDWR | O_CREAT, 0);
  if (IS_ERR(readfile))
      {
          printk(KERN_ERR "open dir %s error.\n", path);
          return -1;
      }
  kernel_read(readfile, testfile, sizeof(struct file), &rpos);
  ```

  


