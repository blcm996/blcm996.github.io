---
title: "OS-Lab：pstree"
tags:
    - linux
    - c++
    - lab
date: "2024-02-04"
thumbnail: "/assets/img/thumbnail/pstree.jpg"
---

# [打印进程树](https://github.com/blcm996/OS_Lab_pstree)

### 适合对象：

Linux&C/C++初学者，掌握了一定c/c++编程基础但还未实际编写应用程序的。

想要学习c++系与系统底层相关技能的入门者。

操作系统课程的学习者

### 实验内容：

参考Linux-bash命令行中pstree的命令参数与效果，复刻一个简版的pstree

（原实验来自NJU2022操作系统课程：[M1: 打印进程树 (pstree) ](https://jyywiki.cn/OS/2022/labs/M1.html)，这是简化改编版本）

### 预期收获：

你将初步了解linux系统内部的文件结构与组织方式，主动学习完成一个支持多个参数的命令行应用程序

当然，也可能你首次接触[STFW](https://lug.ustc.edu.cn/wiki/doc/smart-questions/)和[RTFM](https://lug.ustc.edu.cn/wiki/doc/smart-questions/)，不过现在应该再多一个LWAI（learn with AI）。

了解如何使用Makefile或Cmake等构建工具联合编译C++项目

### 分析与实现:

#### 目标：

程序能够读取命令行输入的指令，并给出相应的输出

**例如：** 

```bash
~/Documents/OS_Lab_pstree$ pstree 
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager───2*[{NetworkManager}]
        ├─VGAuthService
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─acpid
        ├─anacron
        ├─apt.systemd.dai───apt.systemd.dai───apt-get─┬─gpgv
        │                                             ├─3*[http]
        │                                             └─2*[https]
        ├─avahi-daemon───avahi-daemon
...
~/Documents/OS_Lab_pstree$ pstree -p
systemd(1)─┬─ModemManager(916)─┬─{ModemManager}(962)
           │                   └─{ModemManager}(985)
           ├─NetworkManager(847)─┬─{NetworkManager}(945)
           │                     └─{NetworkManager}(947)
           ├─VGAuthService(712)
           ├─accounts-daemon(837)─┬─{accounts-daemon}(939)
           │                      └─{accounts-daemon}(944)
           ├─acpid(838)
           ├─anacron(154364)
           ├─avahi-daemon(841)───avahi-daemon(867)
           ├─bluetoothd(842)
           ├─colord(2112)─┬─{colord}(2115)
           │              └─{colord}(2117)
...
~/Documents/OS_Lab_pstree$ pstree -p -n
systemd(1)─┬─systemd-journal(345)
           ├─vmware-vmblock-(371)─┬─{vmware-vmblock-}(372)
           │                      └─{vmware-vmblock-}(373)
           ├─systemd-udevd(406)
           ├─systemd-oomd(676)
           ├─systemd-resolve(682)
           ├─systemd-timesyn(684)───{systemd-timesyn}(756)
           ├─VGAuthService(712)
           ├─vmtoolsd(714)─┬─{vmtoolsd}(898)
           │               ├─{vmtoolsd}(899)
           │               └─{vmtoolsd}(933)
           ├─accounts-daemon(837)─┬─{accounts-daemon}(939)
           │                      └─{accounts-daemon}(944)
           ├─acpid(838)
           ├─avahi-daemon(841)───avahi-daemon(867)
           ├─bluetoothd(842)
           ├─cron(844)
           ├─dbus-daemon(846)
...
```

显而易见，如果你知道“如何将大象放进冰箱”，那么就一定能看出来我们即将设计的程序可以分为“读取并解析指令输入”、“处理数据”和“打印数据”三个阶段。接下来我就会依次讲解这三个阶段的完成过程。

#### 阶段一：解析指令输入

##### 读取指令

如果你熟悉LinuxC编程，一定了解如何读取命令行输入的文本。最常用的莫过于"scanf()"函数。而如果我们想要程序在启动时就能够接收指令参数，就必须启用main函数的参数列表了。
如下所示

```c++
int main(int argc, char *argv[]) {
        //解析命令行指令参数
        prase_arguements(argc, argv);
```

这样我们可以如下的方式启动程序

```bash
>./pstree.cpp pstree -p
```

我将参数处理函数放在了头文件（prase.h）中。
那么如何处理参数呢？

*这里暂停，对于不解“argc”与“argv[]”的小伙伴，建议自行查询一下相关条目，了解一下它们的基本构成与使用方法。*

##### 解析指令

当然，我们可以对读入的一行参数进行逐个的字符串分析，但那样可能有些过于繁琐。这里我查了一下库函数，找到了一个更方便的解决方案，如下所示：

```c++
void prase_arguements(int argc, char *argv[]) {
	//解析cmd参数
	//校验argv合法性
	for(int i = 1; i < argc; i++) {
		assert(argv[i]);
	}	
	int opt = 0;
	while((opt=getopt_long(argc, argv, "pnV", long_option, NULL))!=-1){
		switch(opt){
			case 0: 
				break;
			case 'p': 
				show_pids = 1;break;
			case 'n': 
				numeric_sort = 1;break;
			case 'V': 
				show_version = 1;break;
			default:
				break;
		}
	}
}
```

这里的getopt_long()函数就是c++函数库<getopt.h>中专门用来处理此类命令行命令参数的库函数。
接着再初始化一些静态变量作为每个可供选择的参数的flag

```c++
int show_version = 0, numeric_sort = 0, show_pids = 0;
static const struct option long_option[]={
	{"show_pids", optional_argument, &show_pids, 1},
	{"numeric_sort", optional_argument, &numeric_sort, 1},
	{"show_version", optional_argument, &show_version, 1},
	{NULL, 0, NULL, 0}
};
```

最后我们只要设计一定逻辑，使得程序读取到指令参数后能够正确的激活对应的flag变量
关于如何使用相关库函数，建议自行搜索或者通过AI学习，这里不再赘述。

#### 阶段二：处理进程信息

接下来的部分大概就是整个实验最难也是最关键的部分了

##### 获取进程信息

不知大家是否听过一句话，叫做“[Linux中，一切皆文件](https://zhuanlan.zhihu.com/p/349354666)”。大意就是指在Linux系统中，所有的一切都可以通过文件的方式访问、管理。这包括硬件设备、进程、套接字等，即使它们本身不是文件，也被抽象成文件。

因此，我们可以直接从系统根目录下的“/proc”目录获得包含当前系统所有进程信息的文件。这里建议大家阅读[博客](https://www.cnblogs.com/DswCnblog/p/5780389.html)的同时自己在linux终端找到**该目录**并**打印**（cat）其中的文件，观察其内容结构是否符合预期。

直接说结论，所有**以数字作为文件名的目录**就是我们要找到目标目录。每个数字就是系统进程的id，目录中的**../stat文件**存放的就是该进程的所有信息。我们只需找到代表其**父进程**和**子进程**的字符串（数字），就可以在逻辑上构建出一棵进程树。当然，别忘了一并读取必要的进程信息。

思路有了，接下来就是实践了

相信细心的你一定能发现，“/proc”目录下不仅有以数字作为文件名的文件，还有很多其他无关文件。所以我们不得不先从寻找所有进程相关文件开始。

```c++
void get_pids(const char *dirName){
        //打开指定目录
        DIR *dir = opendir(dirName);
        assert(dir);
        //依次读取目录并转为long类型来保存
        struct dirent *dirItem = NULL;
        while((dirItem = readdir(dir)) != NULL){
                long pid = dirent_to_pid(dirItem);
                if (pid > 0) insert_pid(pid);
        }
}
long dirent_to_pid(struct dirent* dirItem) {
	assert(dirItem);

	long pid = 0;
    //判断是否为子目录
	if(dirItem->d_type == DT_DIR){
		char *name = dirItem->d_name;
		for(int i = 0; name[i]; i++){
			if(name[i] > '9' || name[i] < '0')
				return -1;
			pid = pid * 10 + name[i] - '0';
		}
		return pid;
	}
	return -1;
}
```

这里我的做法是遍历/proc目录下所有文件与文件夹，将所有纯数字组成的目录由字符串转化为长整形存储到一个vector中。这样我们就获得了一个包含当前所有进程号的vector，以便后续使用

*这部分代码涉及到了<dirent.h>库函数的使用，建议不了解的同学暂停查询一下。*

##### 组织信息结构

刚刚提到，具体包含进程信息的文件是每个进程目录下的“stat”文件，其绝对路径的构成也就是“/proc/<ID>/stat”。因此，我们可以使用vector中保存的进程id信息来拼凑出每个进程的绝对路径，并依次读取其中的信息，具体实现如下：

*建议大家了解“stat”文件中的具体结构与代表信息再来阅读下面的代码*

```c++
void get_info(PTree *pNode,long pid){
	char cmd[30] = {0};
	char pstat[24] = {0};
	long ppid = 0;
	//生成文件路径
	sprintf(pstat, "/proc/%ld/stat", pid);
	//打开文件流并读入到缓冲区
	FILE *fstat = fopen(pstat, "r");
	if(!fstat) { return;}
	//从缓冲区读入进程信息
	fscanf(fstat, "%s (%s %c %ld", pstat, cmd, pstat, &ppid);
	pNode->pid = pid;
	pNode->ppid = ppid;
	//标准化处理
	int len = strlen(cmd);
	pNode->cmd = (char*)malloc(len);
	strcpy(pNode->cmd, cmd);
	pNode->cmd[len - 1] = 0;
	//关闭释放文件流
	fclose(fstat);
}
```

上面的代码还涉及一些文件批处理操作、字符串操作等，不过都是些库函数的常规用法，还是建议大家自行了解。

###### 进程树

树结构的设计很简单，程序打印时需要几种数据，就相应的创建几个变量来存储就好。需要注意的是树的维护，我这里借助了**stl中的map**来动态维护PTree

```c++
struct PTree{
	long pid;
	long ppid;
	std::vector<PTree*> spid;
	char *cmd;
};
//通过map动态维护每个pid对应的进程树节点地址，方便完成插入操作
std::map<int, PTree*> ind;
```

相信大家也注意到了，一个进程可能拥有多个子进程，所以我声明了一个vector类型变量用来存储指向子进程节点的指针。进程树的初始化与子节点的维护代码如下所示：

```c++
//进程树初始化
PTree* init_ptree(){
	std::sort(pids.begin(), pids.end());
	//const int size = pids[pids.size()-1];
	//static PTree index[size];
	PTree *pNode = new PTree();
	//初始化0号进程
	pNode->pid = 0;
	ind[0] = pNode;
	return pNode;
}
//通过vector实现动态插入子节点
void insert_ptree(long pid) {
	PTree *pNode = new PTree();
	//从文件读取进程信息
	get_info(pNode, pid);
	//新建pid与节点地址的映射关系
	ind[pid] = pNode;
	//assert(ind[pNode.ppid] != NULL);
	//找到父节点地址并插入
	ind[pNode->ppid] -> spid.push_back(pNode);
}
```

最后，我们只需要一个建树的方法就可以完成该头文件的设计了。当然，你也可以在主函数中编写这个步骤，虽然可能显得不那么优雅，不过对于一个实验作业，到也无伤大雅了。

这部分代码希望大家自行思考设计，需要参考请克隆本项目。

#### 阶段三：打印一棵树

我们现在已经完整的在内存中构建起了一棵树，但还记得官方pstree指令是如何打印树的吗？

```bash
~/Documents/OS_Lab_pstree$ pstree -p
systemd(1)─┬─ModemManager(916)─┬─{ModemManager}(962)
           │                   └─{ModemManager}(985)
           ├─NetworkManager(847)─┬─{NetworkManager}(945)
           │                     └─{NetworkManager}(947)
           ├─VGAuthService(712)
           ├─accounts-daemon(837)─┬─{accounts-daemon}(939)
...
```

没错，我们现在就是要把这棵逻辑上存放于内存的树给它形象的“画”出来，还要能够根据命令**提供参数**的不同作出不同的“画”。

有一定编程经验的小伙伴一定能看出来，运用递归的思想就可以优雅的设计出这棵树的形体。相信大家都尝试过打印“杨辉三角”吧，这也是同样的道理。不了解的小伙伴可以点击[这里学习](https://blog.csdn.net/m0_52072919/article/details/119032272)如何[打印杨辉三角](http://101.132.172.57/problem.php?id=1088)。

这里给出鄙人的设计思路，仅供参考：

```c++
//几种不同的打印树的逻辑，具体由命令行参数决定
void print_tree(PTree *root,char *pre, int idx){
        if(root == NULL) return;
        //打印行前缀
        if(idx != 0)
                printf("%s", pre);
        //打印行末的进程信息并回车
        if(root->spid.size() == 0)
        {printf("-%s\n", root->cmd);return;}
        //打印该节点信息并遍历打印子节点
        if(root->spid.size() > 0){
                printf("-%s--", root->cmd);
                //更新行前缀
                char *new_pre;
                int len = strlen(pre);
                int lens = len + strlen(root->cmd) + 3;
                new_pre = (char*)malloc(lens);
                strcpy(new_pre, pre);
                for(int i = len; i < lens-1; i++)
                        new_pre[i] = ' ';
                if(root->spid.size() == 1)
                        new_pre[lens-1] = ' ';
                else
                        new_pre[lens-1] = '|';
                //遍历打印子节点，下标i用来使第一个子节点向右打印
                int i = 0;
		std::sort(root->spid.begin(), root->spid.end(), cmp);
                for(auto g:root->spid)
                print_tree(g, new_pre, i), i++;
                return;
        }
        return;
}
```

#### 编译与运行

如果你是第一次执行包含多个头文件的c/c++项目，可以自行学习[makefile的使用](https://makefiletutorial.com/)。

在这个实验项目中，我分别将“**参数解析**”、“**遍历目录**”、“**构建进程树**”以及“**打印进程输**”这几个步骤的代码放到了不同的的源文件中，其中函数的方法写在同名的头文件中，定义则写在源文件里。

对于这种包含多个头文件与源文件的项目，往往需要许多编译步骤，逐步将每个**源文件**编译成**目标文件**、再**链接**到一起才能形成**可执行文件**。有一本书叫做[《程序员的自我修养》](https://gitee.com/younisd/CS-Books/blob/master/%E7%A8%8B%E5%BA%8F%E5%91%98%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%AE%E5%85%BB%E2%80%94%E9%93%BE%E6%8E%A5%E3%80%81%E8%A3%85%E8%BD%BD%E4%B8%8E%E5%BA%93.pdf)，有专门讲解相关知识。

为了方便，我们一般用Makefile或者Cmake等脚本工具来联合编译大型项目，一般的工业项目就连编译指令都是动辄数百行。当然，对于本项目，只需简单的几行指令就可以完成编译了。

```makefile
# Makefile for OS_LAB_PSTREE project

# Compiler and flags
CC = g++
CFLAGS = -Wall -I.

# Source files
SRCS = getpid.cpp pidinfo.cpp prase.cpp treePrint.cpp pstree.cpp

# Object files
OBJS = $(SRCS:.cpp=.o)

# Target binary
TARGET = Pstree

# Default target
all: $(TARGET)

# Linking rule
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

# Compilation rules
%.o: %.cpp
	$(CC) $(CFLAGS) -g -c $<

# Clean rule
clean:
	rm -f $(OBJS) $(TARGET)

```

上面的编译指令就是在AI的帮助下完成的，如果有看不懂的地方建议询问AI。

###### 头文件中的变量

对于C++，要特别注意头文件中变量的声明，尤其是当你想在一个头文件中声明一个全局变量时。

例如本项目中的“vetor<long> pids”变量，由于本程序每次执行**只需遍历一次**当前进程目录，所以我便将其设计为一个全局变量，通过调用“getpids()”方法遍历目录后再将得到的**进程号**保存其中。接着，构建进程树时就可以直接遍历pids中的元素来找到对应进程的**stat文件**。而为了**避免**在多个函数的参数列表中**来回传递参数**，我讲pids的**声明放在了“getpid.h”中**。

而为了**避免**在编译的过程中**重复声明**该变量，就需要将变量声明为**static**。而这样又会遇到新的问题。对于为什么编译器会对直接声明的变量报错，以及声明为static会遇到的问题，大家可以尝试修改并逐步调试，观察在头文件中的static全局变量会有怎样的表现。

在我最新版本的源代码中，使用了**extern**关键字声明该变量，再在**同名源文件**中进行**定义**，就可以比较理想的实现上述需求。“treePrint.h”中的部分变量的声明也是同理。

### 总结

本实验对于只完成过**课程实验**题目/**竞赛**题目编程，未接触过**实际系统底层编程**的同学能够带来较大的帮助。

**独立完成**本实验，就能通过多次**调试代码**，并在**查找与阅读**资源文档等过程中培养**高效自主学习**的本领。



本篇指导文章还有打印树的步骤、调试纠错、防御性编程等相关方面话题未展开讨论，日后若有机会笔者会继续完善。也欢迎阅读至此的各位向我提出见解或疑问：b886866b@163.com;

