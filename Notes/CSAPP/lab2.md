## 1. 实验目的

- 给linux 0.11添加两个系统调用： `iam()` , `whoami()` , 功能如下

- - `int iam(const char* name)` ; 将字符串参数name的内容复制到内核并保存
  - `int whoami(char* name, unsigned int size)` ; 将 `iam()` 保存的名字复制到name指向的用户空间地址，name的大小由size说明**（疑问1，如何说明？）**

- 编写两个用户态测试程序 `iam.c` 和 `whoami.c` 进行测试，输出为：

```
[user/root]# ./iam lizhijun
[user/root]# ./whoami
lizhujun
```



## 2. 实验环境

- Ubuntu虚拟机
- bochs，Linux-0.11



## 3. 实验内容

- 添加三个文件： `who.c` ; `iam.c` ; `whoami.c` 

- - 位于 `linux-0.11/kernel; hdc/usr/rooot` **（疑问3，为啥iam.c和whoami.c要放在这个地方？）** 

- 修改： `unistd.h; system_call.s; sys.h; Makefile` 

- - 修改 `unistd.h` 需使用 `sudo ./mount-hdc` 挂载后进入 `hdc/usr/include` 进行 **（疑问2，为啥需要这样做？）** 
  - `linux-0.11/kernel/; linux-0.11/include/linux; linux-0.11/kernel` 



- who.c 

```
#include <asm/segment.h>
#include <linux/kernel.h>
#include <errno.h>

#define maxlen 23
/*
char为0相当于定义为nul
*/
char kernelname[maxlen + 1] = {0}; 

int sys_iam(const char * name)
{
    printk("sys_iam is runing...\n");
    unsigned int i = 0;
    unsigned int namelen = 0;

    /*
    char数组最后一位为空
    */
    while(get_fs_byte(name + namelen) != '\0')
        ++namelen;
    if(namelen > maxlen){
        errno = EINVAL;
        return -1;
    }
    while(i <= namelen){
        kernelname[i] = get_fs_byte(name + i);
        i++;
    }

    kernelname[i] = '\0';
    return namelen;
}
int sys_whoami(char* name, unsigned int size)
{
    printk("sys_whoami is runing...\n");
    unsigned int i = 0;
    unsigned int namelen = 0;
    while(kernelname[namelen] != '\0')
        ++namelen;
    if(size < namelen){
        errno = EINVAL;
        return -1;
    }
    for(; i <= namelen; ++i){
        put_fs_byte(kernelname[i], name + i);
    }
    return namelen;
}
```

- iam.c 

```
#define __LIBRARY__
#include <unistd.h>
#include <stdio.h>
#incldue 

_syscall1(int, iam, const char*, name);

#define NAMELEN 50
char myname[NAMELEN] = {0};


int main(int argc, char* argv[]){
    int res = 0;
    unsigned int namelen = 0;

    if(argc < 2){
        return -1;
    }else{
        while(argv[1][namelen] != '\0'){
            myname[namelen] = argv[1][namelen];
            ++namelen;
        }
    }
    res = iam(myname);

    if(res == -1)
        printf("systemcall have bug, res: %d, errno:%d\n", res, errono);
    
}
```

- whoami.c

```
#define __LIBRARY__
#include<unistd.h>
#include<stdio.h>
#include<errno.h>

/* 功能:用户态文件,测试系统调用whoami() */

_syscall2(int, whoami, char*, myname, unsigned int, size)

#define SIZE 23

int main(int arg, char ** argv)
{
    char myname[SIZE + 1] = {0};
    unsigned int res = 0;

    res = whoami(myname, SIZE + 1);
    printf("%s\n", myname);
    return 0;
}
```



## 4.实验结果



## 5. 心得体会

- 不愧是哈工大，整个实验看懂花了2小时，写bug花了4小时，改bug花了2小时，但是写完后还是收获很大
- 留了三个疑问，希望以后能弄明白