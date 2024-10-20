# **ls**



### 判断是否有路径参数



![image-20230227194827122](/home/lu/.config/Typora/typora-user-images/image-20230227194827122.png)

optind初值为1,读取一个选项+1（包括选项所带的参数和错误的选项）.每个选项字符后可以跟一个冒号字符（:），表示这个选项带有一个参数。记录当前目录，while(optind<argc)改变工作目录为每个参数（记得读取完返回源目录）。

函数返回读取的选项，错误返回(？)，optind存储每次读取的选项（包括错误的）。

![image-20230227195012822](/home/lu/.config/Typora/typora-user-images/image-20230227195012822.png)

检测到一个选项就把该选项的值（全局变量）变为1。

### 选项-l



![image-20230228002329505](/home/lu/.config/Typora/typora-user-images/image-20230228002329505.png)

​	把readdir()的返回值作为参数传给ls_l()，调用stat（）。

![image-20230227082955187](/home/lu/.config/Typora/typora-user-images/image-20230227082955187.png)

![image-20230228115723775](/home/lu/.config/Typora/typora-user-images/image-20230228115723775.png)

![image-20230228180052660](/home/lu/.config/Typora/typora-user-images/image-20230228180052660.png)

![image-20230228180029632](/home/lu/.config/Typora/typora-user-images/image-20230228180029632.png)

​	把stat结构体中的st_mode和以上常量相与（&），输出文件的类型和权限位。

![image-20230228194554801](/home/lu/.config/Typora/typora-user-images/image-20230228194554801.png)

![image-20230228195122839](/home/lu/.config/Typora/typora-user-images/image-20230228195122839.png)

![image-20230228195157923](/home/lu/.config/Typora/typora-user-images/image-20230228195157923.png)

​	把stat结构体中的st_uid传入getpwuid（）获取输出struct passwd中的pw_name。同理也可获得组名。

​	其余信息也都在stat结构体中,时间用ctime()按格式输出

### 选项-r

在函数内重新调用函数进行递归，即可倒序输出。



### 选项-i

调用stat（），输出struct stat 中的 st_ino。



### 选项-s

调用stat（），输出struct stat 中的 st_blocks。

(我的是512字节一块所以输出时除2)



### 选项-R

遍历完一个目录后，后用rewinddir()再遍历一次如果文件名是“.”或".."时continue。其他的用stat（）获得的st_mode 判断文件类型，如果是目录则chdir（）改变工作目录，递归调用opendir("."),调用完后chdir（".."）再返回上一级目录。



### 选项-a

a=0时，如果文件名为"."或".."或第一个字符为'.'，continue。



### 选项-t

先遍历一遍，得到该目录下一共有几个文件，新建一个结构体包括st_mtime和struct dirent *ent，动态分配相应大小的内存。然后rewinddir（）在遍历一遍，把st_mtime和struct dirent *ent存入结构体数组。用qsort（）给它降序排序。



## 遇到的问题

#### 1.qsort（）

​		我写的qsort() 和cmp（）应该是没问题的，但就是没排序，不得已用网上找的冒泡。（是Linux自带的头文件有问题吗？？）

#### 2.局部变量全局变量名相同

表示选项-i的全局变量i和循环变量i相同，导致不明意义的输出i节点号。

#### 3.malloc（）完之后要free()

不然会段错误 或"malloc(): corrupted top size"。

#### 4.cv后记得改变量名

从别的函数cv过来后变量没有改为该函数内对应的变量名。

#### 5.在chdir（）后应opendir（"."）

改变工作目录后又打开了工作目录的名字 ".."（或是含路径的参数）会输出错误的路径里的内容。

#### 6.qsort()中cmp()的返回值要强转为int型

#### 7.栈溢出，动态分配内存

在给结构体动态分配完内存后还应给每个struct dirent *ent 动态分配内存，不然closedir（）后readdir （）分配的内存就释放了。最后free（）时也要先把每个struct dirent *ent给free了，再free （fil）。



## 以下为代码实现

```c
#include <stdio.h>
#include <dirent.h>
#include <stdlib.h>
#include<string.h>
#include<unistd.h>
#include<sys/stat.h>
#include<sys/types.h>
#include<time.h>
#include<pwd.h>
#include<grp.h>

typedef struct t{
    time_t time;
    struct dirent * Ent;
}file;

int a,l,t,r,i,s,R;

int cmp(const void *a,const void *b);
void ls_R(struct dirent *ent);
int ls_s(struct dirent *ent);
void ls_l(struct dirent *ent);
void ls_read(DIR *dir);
void ls_open(char *dir_path);

int cmp(const void *a,const void *b){
    file *aa=(file *)a;
    file *bb=(file *)b;
    return (int)(bb->time-aa->time);
}

//时间顺序
void ls_read(DIR *dir){
    int mount=0;
    struct dirent *ent; 
    while ((ent = readdir(dir)) != NULL) {
        mount++;
        }
        file *fil=(file *)malloc((mount)*sizeof(file));
        mount=0;
        rewinddir(dir);
        int t1=0;
        struct stat statbuf;
        while((ent = readdir(dir)) != NULL){
            struct dirent *ent_copy = (struct dirent*)malloc(sizeof(struct dirent));
            memcpy(ent_copy, ent, sizeof(struct dirent));
            stat(ent->d_name,&statbuf);
            fil[t1].time=statbuf.st_mtime;
            fil[t1].Ent=ent_copy;
            t1++;
        }
        if(t==1)qsort(fil,t1,sizeof(file),cmp);//cmp()返回值要强制转化为int
        closedir(dir);
        int blocks=0;
        if(r==0){
           for(int i1=0;i1<t1;i1++){
            
        if(a==0){
            if(strcmp(fil[i1].Ent->d_name,".")==0||strcmp(fil[i1].Ent->d_name,"..")==0||fil[i1].Ent->d_name[0]=='.')
            continue;
        }
        if(i==1){//没有选项-i但会在循环到ls.c时//在循环时i会不规律递增//循环变量名和-i选项重叠
            printf("%lu  ",fil[i1].Ent->d_ino);
        }
        if(s==1){
            blocks+=ls_s(fil[i1].Ent);
        }
        if(l==1){
            ls_l(fil[i1].Ent);
        }
        
        printf("%s  ", fil[i1].Ent->d_name);
        if(l==1) printf("\n");
        }
        if(s==1){
            printf("\n总用量：  %d",blocks);
            blocks=0;
        }
        if(R==1){
        for(int i1=0;i1<t1;i1++){
            if(a==0&&fil[i1].Ent->d_name[0]=='.') {
                free(fil[i1].Ent);
                continue;//cv后变量没有改为该函数内对应的变量
            }
            if(strcmp(fil[i1].Ent->d_name,".")==0||strcmp(fil[i1].Ent->d_name,"..")==0){
                free(fil[i1].Ent);
                continue;
            }
            
            //printf("\n\n");
            ls_R(fil[i1].Ent);
            free(fil[i1].Ent);
        }
        }
        }
    
        if(r==1){
            for(int i1=t1-1;i1>=0;i1--){
            if(a==0){
            if(strcmp(fil[i1].Ent->d_name,".")==0||strcmp(fil[i1].Ent->d_name,"..")==0||fil[i1].Ent->d_name[0]=='.')
            continue;
        }
        if(i==1){
            printf("%lu  ",fil[i1].Ent->d_ino);
        }
        if(s==1){
            blocks+=ls_s(fil[i1].Ent);
        }
        if(l==1){
            ls_l(fil[i1].Ent);
        }
        printf("%s  ", fil[i1].Ent->d_name);
        if(l==1) printf("\n");
        }
        if(s==1){
            printf("\n总用量：  %d",blocks);
            blocks=0;
        }
        if(R==1){
        for(int i1=t1-1;i1>=0;i1--){
            if(a==0&&fil[i1].Ent->d_name[0]=='.') {
                free(fil[i1].Ent);
                continue;
            }
            
            if(strcmp(fil[i1].Ent->d_name,".")==0||strcmp(fil[i1].Ent->d_name,"..")==0){
                free(fil[i1].Ent);
                continue;
            }
            
            ls_R(fil[i1].Ent);
            free(fil[i1].Ent);
        }
            
        }
        }
        free(fil); 
        //pause();
}

//打开子目录
//如果是链接，不打开？？？
void ls_R(struct dirent *ent){
    struct stat statbuf;
    stat(ent->d_name,&statbuf);
    if(S_ISDIR(statbuf.st_mode)&&S_ISLNK(statbuf.st_mode)==0){// &&strcmp(ent->d_name,".")!=0&&strcmp(ent->d_name,"..")!=0
        char name[256];
        printf("\n\033[32m%s/%s：\n\033[0m",getcwd(name,256),ent->d_name);
        chdir(ent->d_name);
        ls_open(".");
        chdir("..");
    }
    
}

//块大小
int ls_s(struct dirent *ent){
    struct stat statbuf;
stat(ent->d_name,&statbuf);
printf("%lu  ",statbuf.st_blocks/2);
return statbuf.st_blocks/2;
}

//详细属性
void ls_l(struct dirent *ent){
    struct stat statbuf;
stat(ent->d_name,&statbuf);
        switch (statbuf.st_mode & S_IFMT)
    {
        case S_IFREG:
            printf("-");
            break;
        case S_IFCHR:
            printf("c");
            break;
        case S_IFBLK:
            printf("b");
            break;
        case S_IFDIR:
            printf("d");
            break;
        case S_IFIFO:
            printf("p");
            break;
        case S_IFLNK:
            printf("l");
            break;
        case S_IFSOCK:
            printf("s");
            break;
    }
    if(statbuf.st_mode&S_IRUSR)printf("r");else printf("-");
    if(statbuf.st_mode&S_IWUSR)printf("w");else printf("-");
    if(statbuf.st_mode&S_IXUSR)printf("x");else printf("-");
    if(statbuf.st_mode&S_IRGRP)printf("r");else printf("-");
    if(statbuf.st_mode&S_IWGRP)printf("w");else printf("-");
    if(statbuf.st_mode&S_IXGRP)printf("x");else printf("-");
    if(statbuf.st_mode&S_IROTH)printf("r");else printf("-");
    if(statbuf.st_mode&S_IWOTH)printf("w");else printf("-");
    if(statbuf.st_mode&S_IXOTH)printf("x");else printf("-");
      struct passwd *pw = getpwuid(statbuf.st_uid);
    struct group *gr = getgrgid(statbuf.st_gid);
            printf("  %lu  %s  %s  %ld  %s",statbuf.st_nlink,pw->pw_name,gr->gr_name,statbuf.st_blksize,ctime(&statbuf.st_mtime));
}

void ls_open(char *dir_path){
    DIR *dir;
    if ((dir = opendir(dir_path)) != NULL) {
            ls_read(dir);
    } else {
        perror("打开目录失败");
        exit(EXIT_FAILURE);
    }
}

int main(int argc, char *argv[]) {
    DIR *dir;
    struct dirent *ent;
    char *dir_path = ".";
   int opt,opterr=0;

    while ((opt = getopt(argc, argv, "alRtris")) != -1) {
        switch (opt) {
            case 'a':
                a=1;
                break;
            case 'l':
                l=1;
                break;
            case 'R':
                R=1;
                break;
            case 't':
                t=1;
                break;
            case 'r':
                r=1;
                break;
            case 'i':
                i=1;
                break;
            case 's':
                s=1;
                break;
            case '?':
                printf("Unknown option: %c\n", optopt);
                break;
        }
    }
    char dir_path_yuan[256];
    char dirr[256];
    getcwd(dir_path_yuan,256);//存储原先的地址
    if(argc==optind){
        printf("\033[32m%s:\n\033[0m",getcwd(dirr,256));
                ls_open(".");
                printf("\n");
            }
    while(optind<argc){
        dir_path=argv[optind];
        chdir(dir_path);
        printf("\033[32m%s:\n\033[0m",getcwd(dirr,256));
        
        ls_open(".");//改变工作目录后又打开了工作目录的名字 ".."（含路径的参数）会出错
        chdir(dir_path_yuan);
               printf("\n");
               optind++;
    }
    
               
    return EXIT_SUCCESS;
}
```

