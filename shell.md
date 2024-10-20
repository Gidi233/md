打造一个绝无伦比的 `xxx-super-shell` (`xxx` 是你的名字)，它能实现下面这些功能：

- 实现 管道 (也就是 `|`)
- 实现 输入输出重定向(也就是 `<` `>` `>>`)
- 实现 后台运行（也就是 `&` ）
- 实现 `cd`，要求支持能切换到绝对路径，相对路径和支持 `cd -`
- 屏蔽一些信号（如 `ctrl + c` 不能终止）
- 界面美观
- 开发过程记录、总结、发布在个人博客中

要求：

- 不得出现内存泄漏，内存越界等错误
- 学会如何使用 gdb 进行调试，使用 valgrind 等工具进行检测

#### 

#### Example

```
xxx@xxx ~ $ ./xxx-super-shell
xxx@xxx ~ $ echo ABCDEF
xxx@xxx ~ $ echo ABCDEF > ./1.txt
xxx@xxx ~ $ cat 1.txt
xxx@xxx ~ $ ls -t >> 1.txt
xxx@xxx ~ $ ls -a -l | grep abc | wc -l > 2.txt
xxx@xxx ~ $ python < ./1.py | wc -c
xxx@xxx ~ $ mkdir test_dir
xxx@xxx ~/test_dir $ cd test_dir
xxx@xxx ~ $ cd -
xxx@xxx ~/test_dir $ cd -
xxx@xxx ~ $ ./xxx-super-shell # shell 中嵌套 shell
xxx@xxx ~ $ exit
xxx@xxx ~ $ exit
```

注： 示例请参考 `Bash`、`Zsh` 命令

#### 

#### 语言要求

语言在 C、C++、Go、Rust 中任选

#### 

#### 截止时间

2023-03-25 20：00（UTC+8）

#### 

#### 知识要点

1. 懂得如何使用 shell
2. 理解 shell 原理
3. Linux系统编程：进程控制
4. gdb
5. valgrind



## 一个简易的shell框架

要用到以下函数

![image-20230315063920905](/home/lu/.config/Typora/typora-user-images/image-20230315063920905.png)

![image-20230315064112087](/home/lu/.config/Typora/typora-user-images/image-20230315064112087.png)

![image-20230315190853844](/home/lu/.config/Typora/typora-user-images/image-20230315190853844.png)

![image-20230315193723483](/home/lu/.config/Typora/typora-user-images/image-20230315193723483.png)

在while（1）循环中fork出一个子进程（通过fork（）的返回值判断父进程子进程）调用execvp（），新进程将完全替代子进程（即旧进程在execvp后的指令将被丢弃），父进程调用wait 防止进程混乱。

## 管道

pipe（int fd[2]）先开一个文件句柄数组作为参数，fd[0]为读端，fd[1]为写端，通常由父进程在fork前调用，以进行进程间的信息传输（父子进程须分别关掉相应文件句柄），一般情况下一个管道的读端写端分别只由一个进程持有，以保证信息正常传输（不会出现竞争）.调用dup2(fd[0],STDIN_FILENO),dup2(fd[1],STDOUT_FILENO)，分别将标准输入输出指到管道的写端读端。



## 文件句柄与FILE * 之间的转换

![image-20230321193933174](/home/lu/.config/Typora/typora-user-images/image-20230321193933174.png)



## 信号屏蔽

![image-20230322194624811](/home/lu/.config/Typora/typora-user-images/image-20230322194624811.png)

SIGINT
当用户键入终端中断字符（通常为 Control-C）时，终端驱动程序将发送该信号给前台进
程组。该信号的默认行为是终止进程。

![image-20230322194656693](/home/lu/.config/Typora/typora-user-images/image-20230322194656693.png)

可以自己写函数作为handler参数。

所以屏蔽ctrl+c信号只需在main函数里加上signal(SIGINT, SIG_IGN);



## 后台运行

当你在 shell 中输入一个带有 & 符号的命令时，shell 会创建一个子进程来执行该命令。在这个子进程中，shell  会将标准输入、标准输出和标准错误输出都重定向到 /dev/null，这样就不会影响到当前终端的输入输出。同时，子进程会将自己的进程组 ID  改为与父进程不同，从而与当前终端的进程组脱离关系。

setsid创建一个新进程 让后台运行（子进程）脱离父进程。



## 处理解析命令

char *strtok(char *str, const char *delim)

#### 参数

- **str** -- 要被分解成一组小字符串的字符串。
- **delim** -- 包含分隔符的 C 字符串。（可包含多个字符）

#### 返回值

该函数返回被分解的第一个子字符串，如果没有可检索的字符串，则返回一个空指针。



# 具体实现

先gets( )一次性全部输入，然后调用strtok( ,"|");将输入的字符串分解为一个个命令。

```c
        command[command_amout++]=strtok(sr,"|");
        for(;;command_amout++){
        command[command_amout]=strtok(NULL,"|");
        if(command[command_amout]==NULL) break;
        }
```

然后再shuru（）里再调用strtok(command[i] ,"  ")将命令分为一个个参数。



注：

不能write(STDIN_FILENO,command[i],strlen(command[i])+1); 把读取到的命令一部分一部分以while((ch=getchar())!='\n') scanf(); 的方式读入,不能写入到标准输入（应该和文件权限有关），可以write写到一个临时文件里，再通过freopen(文件名,"r",stdin)把把命令读到缓冲区。



如果command_amout为1（即没有管道），调用shuru（）如果command[0]为cd，调用chdir（），标记已更改路径，（为调用参数-）记录当前路径。如果为exit，break出循环结束。else 调用execute(args)（fork出子进程执行命令）。

如果command_amout大于1，父进程开command_amout-1个管道，然后再开command_amout个子进程（在fork后关掉上一个管道的读写），最后调用command_amout个wait（）。子进程分为第一个最后一个和其余分别保留关闭对应的管道读写端。

最后调用jiancha（）将重定向后的标准输入输出重新指向键盘屏幕。



在shuru（）中检查strtok(command[i] ,"  ")分隔开的每个参数，如果有">",">>","<" ，就截取下一个参数（文件名）输入：fileno(freopen(chongin,"r",stdin))  输出：fileno(freopen(argbuf,"w",stdout)) 追加：fileno(freopen(argbuf,"a",stdout))。（fileno：将FILE *转为系统调用的文件句柄）如果是“&”就将输出重定向到“/dev/null”（伪）。注：要在之前dup（STDIN_FILENO）dup（STDOUT_FILENO）保留初始输入输出的位置（不能int fd=STDIN_FILENO）。



# 代码

```c
#include<unistd.h>
#include<stdio.h>
#include<string.h>
#include<sys/types.h>
#include<sys/wait.h>
#include<fcntl.h>
#include<stdlib.h>
#include<signal.h>
void execute(char *args[]);
int shuru(char *command);
void jiancha();

int set_exit=0,argnum,houtai_set;
int fp0,fp0_set,fp0_yuan,fp1,fp1_yuan,fp1_set,fp_hou,cd_set;
char *args[20];


int main(){
    char sr[100];
    char dir_path_yuan[256];
    char dir_path[256];
    
    signal(SIGINT, SIG_IGN);
    int pid=0,shuru_set;
    int command_amout=0;
    char *command[20];
    int fd[10][2];
    getcwd(dir_path,256);
    strcpy(dir_path_yuan,dir_path);
    while(1){
        shuru_set=0;
        if(cd_set) {
            cd_set=0;
            strcpy(dir_path_yuan,dir_path);
        }
        getcwd(dir_path,256);
        printf("\033[32mljc-shell:\033[31m%s$ \033[0m",dir_path);
        fflush(stdout);
        command_amout=0;
        gets(sr);
        command[command_amout++]=strtok(sr,"|");
        for(;;command_amout++){
            command[command_amout]=strtok(NULL,"|");
            if(command[command_amout]==NULL) break;
        }
        
        if(command_amout>1){//父
        for(int i=0;i<command_amout;i++){//先fork出所有的子进程
            if(i!=command_amout-1) pipe(fd[i]);
            //在父进程输入 获得文件句柄继承给子进程（未）
            pid=fork();
            if(pid){
                if(i>0){
                    close(fd[i-1][0]);
                    close(fd[i-1][1]);
                } 
                if(i==command_amout-1){
                    for(int j=0;j<command_amout;j++) wait(NULL);
                }
            }
            else{//子//子进程中不用检查（把stdin，stdout重定向到标准输入输出）不影响父进程。
             if(i==0){
                close(fd[i][0]);
                dup2(fd[i][1],STDOUT_FILENO);
                shuru(command[i]);

                execute(args);
            }
            else if(i==command_amout-1){
                close(fd[i-1][1]);
                dup2(fd[i-1][0],STDIN_FILENO);
                shuru(command[i]);

                execute(args);
            }else{
                close(fd[i-1][1]);
                close(fd[i][0]);
                dup2(fd[i-1][0],STDIN_FILENO);
                dup2(fd[i][1],STDOUT_FILENO);              
                shuru(command[i]);
                
                execute(args);
            }    
            wait(NULL);
            exit(0);  //祖父    
            }
        }
        }
        else{
            if(shuru_set=shuru(command[0])) continue;
            
            if(strcmp(args[0],"cd")!=0) 
            execute(args);
            else{
                cd_set=1;
                if(strcmp(args[1],"-")==0) chdir(dir_path_yuan);
                else chdir(args[1]);//绝对路径(~)调用失败
            }

        }
        jiancha();
        if(set_exit==1) break;        
    }
}

void jiancha(){
    if(set_exit==1) {//管道去掉检查退出
        for(int i=0;i<20;i++) free(args[i]);
    }
    if(fp0_set) close(fp0),dup2(fp0_yuan,fp0);
    if(fp1_set) close(fp1),dup2(fp1_yuan,fp1);
}

void execute(char *args[]){
    int pid=fork();
    if(pid){
        if(houtai_set) houtai_set=0,printf("[1] %d\n",pid);
        wait(NULL);//后台的话应该不wait？？

    }
    else{
        execvp(args[0],args);//进程被新进程覆盖，之后的指令不运行
        perror("");
        exit(0);
    }
}

int shuru(char *command){
    char ch,*argbuf,*chongin;
    int chongin_set=0;
    argnum=0;
    argbuf=strtok(command," ");
    if(argbuf==NULL){
        return 1;
    }
    do{
        
        if(argnum==0&&strcmp(argbuf,"exit")==0){
        set_exit=1;
        }   
        if(strcmp(argbuf,"&")==0&&fp1_set==0){
            houtai_set=1;
            fp1_yuan=dup(STDOUT_FILENO);
            fp_hou=open("/dev/null",O_RDWR);
            fp1=dup2(fp_hou,STDOUT_FILENO);
            fp1_set=1;
            break;
        }
        if(strcmp(argbuf,"<")==0||strcmp(argbuf,">")==0||strcmp(argbuf,">>")==0) {
            if(strcmp(argbuf,"<")==0){
                chongin=strtok(NULL," ");
                chongin_set=1;
            }
            if(strcmp(argbuf,">")==0){//strtok 区分> >> ？？//指令不带空格
                argbuf=strtok(NULL," ");
                fp1_yuan=dup(STDOUT_FILENO);
                fp1=fileno(freopen(argbuf,"w",stdout));
                fp1_set=1;
            }
            if(strcmp(argbuf,">>")==0){
                argbuf=strtok(NULL," ");
                fp1_yuan=dup(STDOUT_FILENO);
                fp1=fileno(freopen(argbuf,"a",stdout));
                fp1_set=1;
            }
        }
        else{
        args[argnum]=realloc(args[argnum],strlen(argbuf)+1);
        strcpy(args[argnum++],argbuf);            
        }
        argbuf=strtok(NULL," ");
        if(argbuf==NULL) break;
    }while(1);
    args[argnum]=NULL;
    if(chongin_set){//在读取完所有命令后再重定向到标准输入
        fp0_yuan=dup(STDIN_FILENO);
        fp0=fileno(freopen(chongin,"r",stdin));
        fp0_set=1;
    }
    return 0;
}

```



# gdb调试

https://zhuanlan.zhihu.com/p/74897601

https://zhuanlan.zhihu.com/p/297925056

## 常用命令

| 命令名称    | 命令缩写  | 命令说明                                         |
| ----------- | --------- | ------------------------------------------------ |
| run         | r         | 运行一个待调试的程序                             |
| continue    | c         | 让暂停的程序继续运行                             |
| next        | n         | 运行到下一行                                     |
| step        | s         | 单步执行，遇到函数会进入                         |
| until       | u         | 运行到指定行停下来                               |
| finish      | fi        | 结束当前调用函数，回到上一层调用函数处           |
| return      | return    | 结束当前调用函数并返回指定值，到上一层函数调用处 |
| jump        | j         | 将当前程序执行流跳转到指定行或地址               |
| print       | p         | 打印变量或寄存器值                               |
| backtrace   | bt        | 查看当前线程的调用堆栈                           |
| frame       | f         | 切换到当前调用线程的指定堆栈                     |
| thread      | thread    | 切换到指定线程                                   |
| break       | b         | 添加断点                                         |
| tbreak      | tb        | 添加临时断点                                     |
| delete      | d         | 删除断点                                         |
| enable      | enable    | 启用某个断点                                     |
| disable     | disable   | 禁用某个断点                                     |
| watch       | watch     | 监视某一个变量或内存地址的值是否发生变化         |
| list        | l         | 显示源码                                         |
| info        | i         | 查看断点 / 线程等信息                            |
| ptype       | ptype     | 查看变量类型                                     |
| disassemble | dis       | 查看汇编代码                                     |
| set args    | set args  | 设置程序启动命令行参数                           |
| show args   | show args | 查看设置的命令行参数                             |

   用GDB调试多线程程序时，该程序的编译需要添加  **`-lpthread`** 参数。

### 关于多进程的一些命令

1. **info thread**，查看当前调试程序启动了多少个线程，并打印出各个线程信息；
2. **thread 线程编号**，将该编号的线程切换为当前线程；
3. **thread apply 线程编号1 线程编号2 ... command**，将GDB命令作用指定对应编号的线程，可以指定多个线程，若要指定所有线程，用 **all** 替换线程编号；
4. **break location thread 线程编号**，在 **location** 位置设置普通断点，该断点只作用在特定编号的线程上；

### 设置线程锁

   使用GDB调试多线程程序时，默认的调试模式是：**一个线程暂停运行，其他线程也随即暂停；一个线程启动运行，其他线程也随即启动**。但在一些场景中，我们希望只让特定线程运行，其他线程都维持在暂停状态，即要防止**线程切换**，要达到这种效果，需要借助 **set scheduler-locking** 命令。

   命令格式及作用：

- **set scheduler-locking on**，锁定线程，只有当前或指定线程可以运行；
- **set scheduler-locking off**，不锁定线程，会有线程切换；
- **set scheduler-locking step**，当单步执行某一线程时，其他线程不会执行，同时保证在调试过程中当前线程不会发生改变。但如果在该模式下执行 **continue、until、finish** 命令，则其他线程也会执行；
- **show scheduler-locking**，查看线程锁定状态；



(gdb 文件名 -tui )  //有点意思



## 未完成

##### 1.readline库实现历史命令、命令补全、错误命令报错

##### 2.绝对路径‘~’

对参数的第一个字符判断if(args[i][0\]=='~');args[i\]=strcat("/home/用户名",args[i\][1\]);

##### 3.多重重定向

改成只有父子进程，开一个fd[]让父进程shuru（）储存读到需要写入的文件句柄，让子进程输出到一个新建的临时文件，父进程再从文件读取（或者开一个管道分别让子进程写给父进程），再for循环分别write给之前存储的要写入的文件和管道写端。

(印象里之前看到过能同时写入多个文件句柄的函数但想不起来了)



（蓝桥杯就剩一周了，算法还是很suck，先把这些搁置了）



## 问题

##### 1.命令参数中不能带空格（如：awk '{print $1}' 、路径文件夹名带空格）

##### 2.在多管道的子进程再调用execute(args) 即再fork出子进程的子进程 祖父进程会一直卡在wait

（且还不会用gdb调试多进程。）

解决：孙子进程execvp结束了，子进程没有调用exit（）（继续进行shell的while循环）；

