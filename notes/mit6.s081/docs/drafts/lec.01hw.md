# LEC 1 (rtm): Introduction and examples

本节作业：

目录：

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=2 orderedList=false} -->

<!-- code_chunk_output -->

- [阅读echo、grep和rm](#阅读echo-grep和rm)
- [sleep](#sleep)
- [pingpong](#pingpong)
- [](#)

<!-- /code_chunk_output -->

细分目录：

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [阅读echo、grep和rm](#阅读echo-grep和rm)
  - [echo.c](#echoc)
    - [插播：argc和argv](#插播argc和argv)
  - [grep.c](#grepc)
  - [user/rm.c](#userrmc)
- [sleep](#sleep)
  - [kernel/sysproc.c](#kernelsysprocc)
  - [user/user.h](#useruserh)
  - [user/usys.S](#userusyss)
  - [实现sleep.c](#实现sleepc)
- [pingpong](#pingpong)
- [](#)

<!-- /code_chunk_output -->

## 阅读echo、grep和rm

学习如何从命令行获取参数。

### echo.c

user/echo.c

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;

  for(i = 1; i < argc; i++){
    write(1, argv[i], strlen(argv[i]));
    if(i + 1 < argc){
      write(1, " ", 1);
    } else {
      write(1, "\n", 1);
    }
  }
  exit(0);
}
```

#### 插播：argc和argv

关于 c 语言的 `argc` 和 `argv` ，有一篇回答很好：[What does int argc, char *argv[] mean?](https://stackoverflow.com/questions/3024197/what-does-int-argc-char-argv-mean)

```c
#include <iostream>

int main(int argc, char** argv) {
    std::cout << "Have " << argc << " arguments:" << std::endl;
    for (int i = 0; i < argc; ++i) {
        std::cout << argv[i] << std::endl;
    }
}
```

```bash
$ ./test a1 b2 c3
Have 4 arguments:
./test
a1
b2
c3
```

- `argc` 是参数数量加一
- `argv` 是字符串数组，包含可执行文件名

### grep.c

user/grep.c

```c
// Simple grep.  Only supports ^ . * $ operators.

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

char buf[1024];
int match(char*, char*);

void
grep(char *pattern, int fd)
{
  int n, m;
  char *p, *q;

  m = 0;
  while((n = read(fd, buf+m, sizeof(buf)-m-1)) > 0){
    m += n;
    buf[m] = '\0';
    p = buf;
    while((q = strchr(p, '\n')) != 0){
      *q = 0;
      if(match(pattern, p)){
        *q = '\n';
        write(1, p, q+1 - p);
      }
      p = q+1;
    }
    if(m > 0){
      m -= p - buf;
      memmove(buf, p, m);
    }
  }
}

int
main(int argc, char *argv[])
{
  int fd, i;
  char *pattern;

  if(argc <= 1){
    fprintf(2, "usage: grep pattern [file ...]\n");
    exit(1);
  }
  pattern = argv[1];

  if(argc <= 2){
    grep(pattern, 0);
    exit(0);
  }

  for(i = 2; i < argc; i++){
    if((fd = open(argv[i], 0)) < 0){
      printf("grep: cannot open %s\n", argv[i]);
      exit(1);
    }
    grep(pattern, fd);
    close(fd);
  }
  exit(0);
}

// Regexp matcher from Kernighan & Pike,
// The Practice of Programming, Chapter 9.

int matchhere(char*, char*);
int matchstar(int, char*, char*);

int
match(char *re, char *text)
{
  if(re[0] == '^')
    return matchhere(re+1, text);
  do{  // must look at empty string
    if(matchhere(re, text))
      return 1;
  }while(*text++ != '\0');
  return 0;
}

// matchhere: search for re at beginning of text
int matchhere(char *re, char *text)
{
  if(re[0] == '\0')
    return 1;
  if(re[1] == '*')
    return matchstar(re[0], re+2, text);
  if(re[0] == '$' && re[1] == '\0')
    return *text == '\0';
  if(*text!='\0' && (re[0]=='.' || re[0]==*text))
    return matchhere(re+1, text+1);
  return 0;
}

// matchstar: search for c*re at beginning of text
int matchstar(int c, char *re, char *text)
{
  do{  // a * matches zero or more instances
    if(matchhere(re, text))
      return 1;
  }while(*text!='\0' && (*text++==c || c=='.'));
  return 0;
}
```

### user/rm.c

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    fprintf(2, "Usage: rm files...\n");
    exit(1);
  }

  for(i = 1; i < argc; i++){
    if(unlink(argv[i]) < 0){
      fprintf(2, "rm: %s failed to delete\n", argv[i]);
      break;
    }
  }

  exit(0);
}
```

原来 `rm` 是用 `unlink` 做的？

## sleep

See `kernel/sysproc.c` for the xv6 kernel code that implements the sleep system call (look for `sys_sleep`), `user/user.h` for the C definition of sleep callable from a user program, and `user/usys.S` for the assembler code that jumps from user code into the kernel for sleep.

### kernel/sysproc.c

```c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

看起来像是一个锁，并且 `sleep(&ticks, &tickslock)` 是用来改变 `ticks` 这些信息的。

没看懂，再往下看看。

### user/user.h

```c
int sleep(int);
```

### user/usys.S

`usys.S` 是 `usys.pl` 生成的，得 `make qemu` 才能看到。

```S
# generated by usys.pl - do not edit
#include "kernel/syscall.h"
...
.global sleep
sleep:
 li a7, SYS_sleep  # 将 SYS_sleep 的低 6 位取出写入到 a7 中
 ecall             # Environment Call
 ret
...
```

### 实现sleep.c

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if (argc < 2){
    fprintf(2, "Usage: sleep TIME\n");
    exit(1);
  }

  sleep(atoi(argv[1]));

  exit(0);
}
```

**Make sure main calls exit() in order to exit your program.**

注意别 `return` ，我们要的是退出进程，而非返回到父进程。

## pingpong

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int p0[2], p1[2];
  char buf[1] = "0";

  pipe(p0);
  pipe(p1);

  if (fork() == 0){
    close(p1[0]);
    close(p0[1]);
    read(p0[0], buf, 1);
    printf("%d: received ping\n", getpid());
    write(p1[1], buf, 1);
    close(p1[1]);
    close(p0[0]);
  } else {
    close(p0[0]);
    close(p1[1]);
    write(p0[1], buf, 1);
    read(p1[0], buf, 1);
    printf("%d: received pong\n", getpid());
    wait(0);
    close(p0[1]);
    close(p1[0]);
  }

  exit(0);
}
```

## 