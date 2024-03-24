# Lab 1

## task-1 sleep

实现一个`sleep` 函数，调用格式为`sleep [time]` , 由于我们可以直接调用`system call`, 所以比较简单。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc,char *argv[]) 
{
    if(argc == 1) 
    {
        printf("sleep need one argument\n");
        exit(1);
    }
    uint time = atoi(argv[1]); //把字符串转为整数
    sleep(time);
    printf("(nothing happens for a little while)");
    exit(0);
}
```

测评方式: 在`xv6-labs-2020` 文件夹下输入:

````shell
./grade-lab-util sleep
````

## system call

- `fork`: 创建一个子进程，其在父进程和子进程中均返回，在父进程中返回该子进程的`PID`, 在子进程中返回`0`.
- `exit`:  使调用它的进程停止，并回收相关资源，参数为`0`代表成功，参数为`1`代表失败
- `wait`: 返回当前进程一个结束的或被杀死的进程的`PID`, 其参数为一个指针，并会把该子进程的`exit status` 赋给该指针
- `exec`: 执行一个可执行文件

## task -2 pingpong

任务要求： 使用两个`pipe`, 父进程首先向子进程发送一个byte，子进程收到后打印`"<pid>: received ping"`, 然后向父进程发送一个byte，并`exit` ,  父进程收到后打印`"<pid>: received pong"`, 然后`exit`.

注意：我们需要及时关闭`pipe`中不被用到的`file descriptor` ，教材里说得很清楚了。

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]){
    char buf = 'a';
    int pipe1[2]; //从父到子
    int pipe2[2]; // 从子到父
    pipe(pipe1);
    pipe(pipe2);
    int pid = fork();
    int exit_status = 0;

    if (pid<0){
        close(pipe1[0]);
        close(pipe2[0]);
        close(pipe1[1]);
        close(pipe2[1]);
        fprintf(2, "fork() error!\n");
        exit(1);
    }

    else if(pid == 0){
        close(pipe1[1]); //子进程只用得到pipe1的读
        close(pipe2[0]); // 子进程只用得到pipe2的写
        //子进程读
        if (read(pipe1[0],&buf,sizeof(char))!=sizeof(char)){
            fprintf(2, "child read() error!\n");
            exit_status = 1;
        }
        else {
            fprintf(1, "%d: received ping\n", getpid());
        }
        //子进程写
        if (write(pipe2[1], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child write() error!\n");
            exit_status = 1;
        }

        close(pipe1[0]);
        close(pipe2[1]);
        exit(exit_status);
    }
    //父进程
    else{
        close(pipe1[0]); //父进程只用得到pipe1的写
        close(pipe2[1]); //子进程只用得到pipe2的读
        //父进程写
        if (write(pipe1[1], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent write() error!\n");
            exit_status = 1;
        }
        //父进程读
        if (read(pipe2[0], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent read() error!\n");
            exit_status = 1; //标记出错
        } else {
            fprintf(1, "%d: received pong\n", getpid());
        }
        exit(exit_status);
    }

}
```

## task - 3 primes

任务要求： 并发编程版本的素数筛法

第一个进程先放入`2~35`, 每次先取出前一个进程的第一个数放入第二个进程，第二个进程打印这个数(这个数一定是素数)， 然后循环第一个进程里的数，如果这个数不能被第一个数整除，那么将其放入第二个进程，这样不断进行下去，最终我们可以打印出所有的素数。

```c
#include "kernel/types.h"
#include "user/user.h"

void passdata(int receive_fd){
    int first_num = 0;
    int is_receive = 0;
    int cur_num = 0;
    int cur_pipe[2];
    while (1){

        int read_bytes = read(receive_fd,&cur_num,4);
        if (read_bytes == 0){ //没有读到输入
            close(receive_fd);
            if (is_receive){ //本进程从上一个进程读取了数
                close(cur_pipe[1]);
                wait((int *) 0); //等待子进程
            }
            //否则直接终止
            exit(0);
        }
    
        else{ //前一个进程的数还没读完

            if (first_num == 0){
                first_num = cur_num;
                printf("prime %d\n", first_num);

            }
            
            if (cur_num % first_num!=0){
                if (!is_receive){ //此时接收到第二个数
                    pipe(cur_pipe);
                    is_receive = 1;
                    int next = fork();

                
                    if (next == 0){
                        close(cur_pipe[1]);
                        close(receive_fd);
                        passdata(cur_pipe[0]);
                    }
                

                    else{
                        close(cur_pipe[0]);
                    }

                }

                write(cur_pipe[1],&cur_num,4);

            }
    }
}
}

int main(int argc, char *argv[]){
    int pipe1[2];
    pipe(pipe1);
    for (int i = 2; i<=35;i++){
        write(pipe1[1],&i,4);
    }
    close(pipe1[1]);
    passdata(pipe1[0]);
    exit(0);
}
```

## task -4 find

要想完成本任务，我们需要完全理解`ls.c`

```c
char *fmtname(char *path){
	static char buf[DIRSIZ+1];
	char *p;
	for (p = path+strlen(path);p>=path && *P!='/';p--)
	;
	p++;
	
	if (strlen(p)>=DIRSIZ) return p;
	memmove(buf,p,strlen(p));
	memset(buf+strlen(p), '', DIRSIZ - strlen(p));
	return buf;
	
}
```

首先我们传入一个路径，然后我们把p定位到最后一个`/`的那个文件名或文件夹名，如果`p`的长度大于`DIRSIZ`, 直接返回`p`, 否则将其扩展到`DIRSIZ+ 1`这么长，然后返回。

```c
void
ls(char *path)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){ //打开path
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){ //把path的信息传给st
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){ //判断该path的类型
  case T_FILE: //如果path代表一个文件，直接打印并退出
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR: //如果path是一个文件夹
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path); //把path复制给buf.
    p = buf+strlen(buf); 
    *p++ = '/'; //在buf末尾加上'/'
    while(read(fd, &de, sizeof(de)) == sizeof(de)){ //每次从fd中读一个de
      if(de.inum == 0) //忽略de.inum=0的de
        continue;
      memmove(p, de.name, DIRSIZ); //把de.name拿给p,也就是buf的末尾,buf变成了新的目录
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){//把buf的信息拿给st,
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}
```

```c
int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    ls(".");
    exit(0);
  }
  for(i=1; i<argc; i++)
    ls(argv[i]);
  exit(0);
}

```

- ls(*path)从传入的参数中获取路径名称

- - 创建缓冲区，一些存储读取的信息的变量（包括一些结构体）

  - 用open()打开路径path，并将文件句柄保存在fd中

  - - 弹出读取过程中的报错

  - 用fstat从句柄中读取当前路径的stat，并保存在st中

  - - 弹出读取过程中的报错

  - 对st.type进行判断，进入switch结构

  - - 当st.type为T_FILE即文件时

    - - 输出当前st的信息
      - break

    - 当st.type为T_DIR即目录时

    - - 读取目录并且将其放入到buf中，用一个活动的指针p去对buf中的内容进行拼接和修饰操作

      - 循环体while，判断条件为是否成功从句柄fd中读取dirent结构（目录层）

      - - 忽略de.inum==0的项
        - 拼接de.name到buf末尾，获得fd指向的目录下的一个文件完整路径
        - stat访问完整路径，存入st
        - 输出st

      - break

大概就是这么一个流程。

这样的话，我们`find.c`的思路就比较清晰了:

在`ls.c`中，我们的路径如果表示一个文件夹，我们就输出其内部的所有东西的名字(包括文件夹)。而现在，如果我们遇到文件，需判断其名字与目标文件是否相同， 相同便输出其路径，如果遇到了文件夹，不能输出其名字了，应该以这个文件夹为起点递归地进行下去，知道找到了相同名字的文件。

```c
 #include "kernel/types.h"
 #include "kernel/stat.h"
 #include "kernel/fs.h"
 #include "user/user.h"

char* GetFileName(char *path){ //找到当前路径的最后一个/后面的那串名字
    char *p;

    for(p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    return p;
}
void find(char *path, char* target){
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }
    switch (st.type)
    {
    case T_FILE: //输入的path是一个文件，那么我们直接将其和我们的目标文件相比较。
        if(strcmp(GetFileName(path), target) == 0){
            printf("%s\n", path);
        
        }
        break;
    
    case T_DIR: //输入的path是一个文件夹
        if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
            printf("find: path too long\n");
            break;
        }
    
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/'; //在buf的屁股后面加上'/'

        while (read(fd, &de, sizeof(de)) == sizeof(de)) //读取该文件夹里面的de
        {
            if(de.inum == 0) //跳过de = 0的de
                continue;
            if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0){ //跳过"." 和 ".."
                continue;
            }
            memmove(p, de.name, DIRSIZ); //把de.name接到buf的后面，构成新的目录
            p[DIRSIZ] = 0;
            if(stat(buf, &st) < 0){
                printf("find: cannot stat %s\n", buf);
                continue;
            }
            find(buf, target);//递归
        }
        break;
    }
    close(fd);
}

int main(int argc, char *argv[]){
    if(argc < 3){
        printf("need 2 dirctory's name.\n");
        exit(1);
    }
    for(int i = 1; i < argc; i++){
        find(argv[i], argv[i+1]);
    }
    exit(0);
}
```

## task -5 xargs

思路：主进程一直从标准输入中读，读到`\n` 后， `fork`一个子进程，然后执行，此时主进程等待子进程的结束，然后继续读。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

#define BUF_SIZE 512

int main(int argc, char* argv[]) {
    if (argc < 2) {
        fprintf(2, "Usage: xargs command\n");
        exit(1);
    }
    char* args[MAXARG+1];
    int index = 0;
    for (int i=1; i<argc; ++i) {
        args[index++] = argv[i];
    }

    char buf[BUF_SIZE];
    char *p = buf;
    while (read(0, p, 1) == 1) {
        if ((*p) == '\n') {
            *p = 0;
            int pid;
            if ((pid = fork()) == 0) {
                // child
                // exec only one arg
                args[index] = buf;
                exec(argv[1], args);
                fprintf(2, "exec %s failed\n", argv[1]);
                exit(0);
            }
            // parent
            wait(0);
            p = buf;
        } else {
            ++p;
        }
    }
    exit(0);
}
```

