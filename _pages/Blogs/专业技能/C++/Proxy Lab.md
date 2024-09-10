---
title: "实验：代理服务器（补偿更新中）"
tags:
    - c++
    - linux
    - Cache
    - web
date: "2023-07-27"
thumbnail: "/assets/img/thumbnail/book.jpg"
bookmark: true
---

# 前言：

该实验来自黑书[《深入理解计算机系统》](https://hansimov.gitbook.io/csapp)。它很好的从代码实践的角度讲解了计算机系统常用知识。陈皓曾这样评价这本书：“为程序员描述计算机系统的实现细节，帮助其在大脑中构造一个层次型的计算机系统。从最底层的数据在内存中的表示到流水线指令的构成，到虚拟存储器，到编译系统，到动态加载库，到最后的用户态应用。通过掌握程序是如何映射到系统上，以及程序是如何执行的，你能够更好地理解程序的行为为什么是这样的，以及效率低下是如何造成的。”

虽然本书所设计的实验距离当下主流的应用开发还有一段距离，但他就好比手动挡的汽车驾照，虽然一般大家都开自动挡的车，但了解如何驾驶手动挡的你可以在自动系统失灵时，仍能够驱动项目运行起来。

## 实验简介

实现一个并发缓存（concurrent caching）的 Web 代理，该代理位于本地的浏览器和其他万维网之间。该实验让学生接触到有趣的网络编程世界，并将课程中的许多概念联系在一起，例如字节序（byte ordering）、缓存、进程控制、信号、信号处理、文件 I/O、并发和同步

*注：本实验项目主要通过C语言编写，面向过程的展示了一个最基础的网络服务器应有的各项功能与逻辑，其次注重实现模拟Cache的算法实现。*

## 实验内容

### Part 1：处理代理请求

代理请求其实与普通的web请求类似，只不过请求头会额外包含一个链接，指向目的服务器的资源。如果不了解普通的web请求有哪几类，分别有哪几部分，可以先自行学习了解。（注：请注意查看实验要求，提前设想程序要处理哪几种web请求，具体把控哪些细节。建议结合各种请求的作用效果思考）

#### 建立连接

这是每个网络服务器都会涉及的最基础部分功能，也是我个人做项目首次接触，因此会在这篇文章详细描述一下具体过程。

```c
    //socket初始化操作
	int listenfd, connfd, port, clientlen;
    struct sockaddr_in clientaddr;
    pthread_t pid;
 
    //这里涉及命令行参数的处理，可跳过自行学习
    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(1);
    }
    port = atoi(argv[1]);
    //打开命令函参数所指定的链接
    listenfd = Open_listenfd(port);
    while (1) {
        //获取客户端链接情况
        clientlen = sizeof(clientaddr);
        //这里的accpet方法做了一层错误处理的包装，具体可查看csapp.h头文件
        connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
        //同理做了一层包装，初学者建议了解一下这两个方法中参数列表每个参数的作用
        //这里新建线程处理新来的链接
        Pthread_create(&pid, NULL, thread, (void*)&connfd);   
    }
```

建议初学者查阅文档了解以上代码中的变量都将保存哪些信息，也可直接调试查看这些变量的实际输出，并持续关注后续代码再次调用该变量时具体用到了哪些数据，这样你将对服务器数据处理的过程有一个更加实际的了解。

###### 多线程处理

```c
void thread(void* v){
    int fd = *(int*)v;
    //线程分离，由操作系统自动完成线程终结后的资源回收
    Pthread_detach(pthread_self());
    doit(fd);
    Close(fd);
}
```

#### 解析请求头

**首先，我们要从链接中取出收到的数据**

```c
void doit(int fd) 
{
    int is_static;
    struct stat sbuf;
    char buf[MAXLINE], method[MAXLINE], uri[MAXLINE], version[MAXLINE];
    char filename[MAXLINE], cgiargs[MAXLINE];
    rio_t rio;
  
    //这里借助了包装过的rio作为连接数据的缓冲区使用
    /* Read request line and headers */
    //首先将fd与rio绑定并初始化
    Rio_readinitb(&rio, fd);
    //将rio绑定的数据写入buf
    Rio_readlineb(&rio, buf, MAXLINE);
    //从buf按照格式中依次读取method、uri、version数据
    sscanf(buf, "%s %s %s", method, uri, version);    
    //服务器不处理GET以外的方法
    if (strcasecmp(method, "GET")) {                     
       clienterror(fd, method, "501", "Not Implemented",
                "Tiny does not implement this method");
        return;
    }
    
    read_requesthdrs(&rio);
    is_static = parse_uri(uri, filename, cgiargs);
```

```c
void read_requesthdrs(rio_t *rp) 
{
    char buf[MAXLINE];
	//循环读取请求头内容并打印，可以将打印信息输入到日志器中，记录客户端连接信息
    Rio_readlineb(rp, buf, MAXLINE);
    while(strcmp(buf, "\r\n")) {          //line:netp:readhdrs:checkterm
	Rio_readlineb(rp, buf, MAXLINE);
	printf("%s", buf);
    }
    return;
}
```

代理请求的uri示例：http://proxy.serverB.com:8080/http://serverC.com/resource

```c
int parse_uri(char *uri, char *filename, char *cgiargs) 
{
    char *ptr;
	
    if (!strstr(uri, "cgi-bin")) {  /* Static content */ 
        strcpy(cgiargs, "");                             
        strcpy(filename, ".");                           
        strcat(filename, uri); 

        if (uri[strlen(uri)-1] == '/')                   
            strcat(filename, "index.html");               
        	return 1;
        }
        else {  /* Dynamic content */                        
            ptr = index(uri, '?');                           
            if (ptr) {
                strcpy(cgiargs, ptr+1);
                *ptr = '\0';
        }
	else 
	    strcpy(cgiargs, "");                         
        strcpy(filename, ".");                           
        strcat(filename, uri);                           
        return 0;
    }
}
```



## Part 2：处理请求资源

处理文件资源请求：查找指定文件或生成反馈结果并返回

```c
    

	if (stat(filename, &sbuf) < 0) {                     
        clienterror(fd, filename, "404", "Not found",
                "Tiny couldn't find this file");
        return;
    }                                                    

    if (is_static) { /* Serve static content */          
	if (!(S_ISREG(sbuf.st_mode)) || !(S_IRUSR & sbuf.st_mode)) { 
	    clienterror(fd, filename, "403", "Forbidden",
			"Tiny couldn't read the file");
	    return;
	}
	serve_static(fd, filename, sbuf.st_size);        
    }
    else { /* Serve dynamic content */
	if (!(S_ISREG(sbuf.st_mode)) || !(S_IXUSR & sbuf.st_mode)) { 
	    clienterror(fd, filename, "403", "Forbidden",
			"Tiny couldn't run the CGI program");
	    return;
	}
	serve_dynamic(fd, filename, cgiargs);           
    }
}
```



### Part 3：维护缓存



### Part 4：测试与问题处理



