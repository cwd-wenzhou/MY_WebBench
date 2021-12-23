# WebBench
增加了一个新的选项-k,用来支持对HTTP长连接的测试。用法与原来的没什么区别。
默认不加k为短连接，加k的时候请与-2(http11)配合使用。

## 使用：

	sudo make && sudo make install PREFIX=your_path_to_webbench
  
## 命令行选项：



| 短参        | 长参数           | 作用   |
| ------------- |:-------------:| -----:|
|-f     |--force                |不需要等待服务器响应               | 
|-r     |--reload               |发送重新加载请求                   |
|-t     |--time <sec>           |运行多长时间，单位：秒"            |
|-p     |--proxy <server:port>  |使用代理服务器来发送请求	    |
|-c     |--clients <n>          |创建多少个客户端，默认1个"         |
|-9     |--http09               |使用 HTTP/0.9                      |
|-1     |--http10               |使用 HTTP/1.0 协议                 |
|-2     |--http11               |使用 HTTP/1.1 协议                 |
|-k    |使用 Keep-Alive长连接进行测试|
|       |--get                  |使用 GET请求方法                   |
|       |--head                 |使用 HEAD请求方法                    |
|       |--options              |使用 OPTIONS请求方法               |
|       |--trace                |使用 TRACE请求方法                 |
|-?/-h  |--help                 |打印帮助信息                       |
|-V     |--version              |显示版本号                         |


# webbench

webbench总共就是三个文件，makefile,socket.c,webbench.c

# makefile

完成编译，`sudo make install`安装到 `/usr/local/bin`目录下面。

其实 make install命令就是把可执行文件拷贝到/bin目录下面。

```c
INSTALL(1)                       User Commands                      INSTALL(1)

NAME
       install - copy files and set attributes

SYNOPSIS
       install [OPTION]... [-T] SOURCE DEST
       install [OPTION]... SOURCE... DIRECTORY
       install [OPTION]... -t DIRECTORY SOURCE...
       install [OPTION]... -d DIRECTORY...

DESCRIPTION
       This  install  program copies files (often just compiled) into destina‐
       tion locations you choose.  If you  want  to  download  and  install  a
       ready-to-use package on a GNU/Linux system, you should instead be using
       a package manager like yum(1) or apt-get(1).
```

## phony：

PHONY定义伪目标的命令一定会被执行，下面尝试分析这种优点的妙处。

1、如果我们指定的目标不是创建目标文件，而是使用makefile执行一些特定的命令，例如：

`clean:rm *.o temp`我们希望，只要输入"make clean"后，"rm *.o temp"命令就会执行。但是，当当前目录中存在一个和指定目标重名的文件时，例如clean文件，结果就不是我们想要的了。输入"make clean"后，"rm *.o temp" 命令一定不会被执行。

解决的办法是，将目标clean定义成伪目标就成了。无论当前目录下是否存在"clean"这个文件，输入"make clean"后，"rm *.o temp"命令都会被执行。

注意：这种做法的带来的好处还不止此，它同时提高了make的执行效率，因为将clean定义成伪目标后，make的执行程序不会试图寻找clean的隐含规则。

# Socket.c

Sokcet.c封装了对于目标网站的TCP套接字的构造，其中Socket函数用于获取连接目标网站TCP套接字.使用了gethostbyname来进行dns查询。使得webbench支持域名测试与直接ip测试。

# webbench.c

小函数：

alarm_handler :作为超时信号的回调函数，设置 `timerexpired` 全局变量为1；

usage:用于向控制台打印帮助信息。

核心就是三个函数`benchcore` ,`build request` ,`bench`

## build_request

根据传入的url以及一些全局设置，把请求报文的字符串拼接出来。

![Untitled](webbench%20408cdfc4ccd44247ad802140e4410cad/Untitled.png)

```c
//把请求报文给拼出来
void build_request(const char *url)
{
   char tmp[10];
   int i;

   bzero(host, MAXHOSTNAMELEN);
   bzero(request, REQUEST_SIZE);

   if (force_reload && proxyhost != NULL && http10 < 1)
      http10 = 1;
   if (method == METHOD_HEAD && http10 < 1)
      http10 = 1;
   if (method == METHOD_OPTIONS && http10 < 2)
      http10 = 2;
   if (method == METHOD_TRACE && http10 < 2)
      http10 = 2;

//根据method，写下请求方法
   switch (method)
   {
   default:
   case METHOD_GET:
      strcpy(request, "GET");
      break;
   case METHOD_HEAD:
      strcpy(request, "HEAD");
      break;
   case METHOD_OPTIONS:
      strcpy(request, "OPTIONS");
      break;
   case METHOD_TRACE:
      strcpy(request, "TRACE");
      break;
   }

//写空格
   strcat(request, " ");

//判断输入的url，非http的抛弃，过长的抛弃
   if (NULL == strstr(url, "://"))
   {
      fprintf(stderr, "\n%s: is not a valid URL.\n", url);
      exit(2);
   }
   if (strlen(url) > 1500)
   {
      fprintf(stderr, "URL is too long.\n");
      exit(2);
   }
   if (proxyhost == NULL)
      if (0 != strncasecmp("http://", url, 7))
      {
         fprintf(stderr, "\nOnly HTTP protocol is directly supported, set --proxy for others.\n");
         exit(2);
      }
   /* protocol/host delimiter */
   i = strstr(url, "://") - url + 3; //i储存url的首地址
   /* printf("%d\n",i); */

   if (strchr(url + i, '/') == NULL)
   {
      fprintf(stderr, "\nInvalid URL syntax - hostname don't ends with '/'.\n");
      exit(2);
   }
   if (proxyhost == NULL)
   {
      /* get port from hostname */
      if (index(url + i, ':') != NULL &&
          index(url + i, ':') < index(url + i, '/'))
      {
         strncpy(host, url + i, strchr(url + i, ':') - url - i);
         bzero(tmp, 10);
         strncpy(tmp, index(url + i, ':') + 1, strchr(url + i, '/') - index(url + i, ':') - 1);
         /* printf("tmp=%s\n",tmp); */
         proxyport = atoi(tmp);
         if (proxyport == 0)
            proxyport = 80;
      }
      else
      {
         strncpy(host, url + i, strcspn(url + i, "/"));
      }
      // printf("Host=%s\n",host);
      strcat(request + strlen(request), url + i + strcspn(url + i, "/"));//写下请求url
   }
   else
   {
      // printf("ProxyHost=%s\nProxyPort=%d\n",proxyhost,proxyport);
      strcat(request, url);
   }
//写下http协议及其版本
   if (http10 == 1)
      strcat(request, " HTTP/1.0");
   else if (http10 == 2)
      strcat(request, " HTTP/1.1");
   strcat(request, "\r\n");
   if (http10 > 0)
      strcat(request, "User-Agent: WebBench " PROGRAM_VERSION "\r\n");
   if (proxyhost == NULL && http10 > 0)
   {
      strcat(request, "Host: ");
      strcat(request, host);
      strcat(request, "\r\n");
   }
   if (force_reload && proxyhost != NULL)
   {
      strcat(request, "Pragma: no-cache\r\n");
   }
   if (http10 > 1)
      strcat(request, "Connection: close\r\n");
   /* add empty line at end */
   if (http10 > 0)
      strcat(request, "\r\n");
   // printf("Req=%s\n",request);
}
```

## bench.c

```c
/* vraci system rc error kod */
static int bench(void)
{
   int i, j, k;
   pid_t pid = 0;
   FILE *f;

//测试一下socket能不能成功连接，如果不能直接结束程序并报告错误原因
   /* check avaibility of target server */
   i = Socket(proxyhost == NULL ? host : proxyhost, proxyport);
   if (i < 0)
   {
      fprintf(stderr, "\nConnect to server failed. Aborting benchmark.\n");
      return 1;
   }
   close(i);
   /* 创建父子进程通信的管道*/
   if (pipe(mypipe))
   {
      perror("pipe failed.");
      return 3;
   }

   /* not needed, since we have alarm() in childrens */
   /* wait 4 next system clock tick */
   /*
  cas=time(NULL);
  while(time(NULL)==cas)
        sched_yield();
  */

   /* fork childs */
//fork clients次，创建这么多个子进程
   for (i = 0; i < clients; i++)
   {
      pid = fork();
      if (pid <= (pid_t)0)
      {
         /* child process or error*/
/*关键：子进程在创建之初睡一下，使父进程可以把所有子进程一口气创建完。
不然可能出现，最早创建的子进程都跑完了，后面的进程还没创建。
达不到高并发亮的测试效果*/
         sleep(1); /* make childs faster */
         break;
      }
   }

   if (pid < (pid_t)0)
   {
      fprintf(stderr, "problems forking worker no. %d\n", i);
      perror("fork failed.");
      return 3;
   }

   if (pid == (pid_t)0)
   {
      /* I am a child */
//调用benchcore进行测试
      if (proxyhost == NULL)
         benchcore(host, proxyport, request);
      else
         benchcore(proxyhost, proxyport, request);

      /* write results to pipe */
      f = fdopen(mypipe[1], "w");//打开管道
      //printf("mypipe[1]=%d\n",mypipe[1]);
      if (f == NULL)
      {
         perror("open pipe for writing failed.");
         return 3;
      }
//把测试结果通过管道写给父进程
      /* fprintf(stderr,"Child - %d %d\n",speed,failed); */
      fprintf(f, "%d %d %d\n", speed, failed, bytes);
      fclose(f);
      return 0;
   }
   else
   {
//父进程，打开管道
      f = fdopen(mypipe[0], "r");
      if (f == NULL)
      {
         perror("open pipe for reading failed.");
         return 3;
      }
///设置管道流为无缓冲
//_IONBF	无缓冲：不使用缓冲。每个 I/O 操作都被即时写入。buffer 和 size 参数被忽略。
      setvbuf(f, NULL, _IONBF, 0);
      speed = 0;
      failed = 0;
      bytes = 0;
//测试时有时候webbench卡住，就应该是在这个while1循环里卡住的。
      while (1)
      {
/*
fscanf 函数原型为 int fscanf(FILE * stream, const char * format, [argument...]); 
其功能为根据数据格式(format)，从输入流(stream)中读入数据，
存储到argument中，遇到空格和换行时结束。
*/
         pid = fscanf(f, "%d %d %d", &i, &j, &k);
         if (pid < 2)
         {
            fprintf(stderr, "Some of our childrens died.\n");
            break;
         }
         speed += i;
         failed += j;
         bytes += k;
         /* fprintf(stderr,"*Knock* %d %d read=%d\n",speed,failed,pid); */
         if (--clients == 0)
            break;
      }
      fclose(f);

      printf("\nSpeed=%d pages/min, %d bytes/sec.\nRequests: %d susceed, %d failed.\n",
             (int)((speed + failed) / (benchtime / 60.0f)),
             (int)(bytes / (float)benchtime),
             speed,
             failed);
   }
   return i;
}
```

## benchcore.c

每一个子进程在时间到期前，都在不停的创建socket，连接服务器，进行http请求并读取返回。成功则记录byte和speed。失败则fail++；

```c
void benchcore(const char *host, const int port, const char *req)
{
   int rlen;
   char buf[1500];
   int s, i;
   struct sigaction sa;

   /* setup alarm signal handler */
//设置一个时钟回调
   sa.sa_handler = alarm_handler;
   sa.sa_flags = 0;
   if (sigaction(SIGALRM, &sa, NULL))
      exit(3);
   alarm(benchtime);

   rlen = strlen(req);
nexttry:
   while (1)
   {
//超时返回
      if (timerexpired)
      {
         if (failed > 0)
         {
            /* fprintf(stderr,"Correcting failed by signal\n"); */
            failed--;
         }
         return;
      }
//连接socket
      s = Socket(host, port);
      if (s < 0)
      {
//创建socket有问题，计入fail，再来一次
         failed++;
         continue;
      }
//向socket写build_request里面拼好的request
      if (rlen != write(s, req, rlen))
      {
//读出了问题，计入fail，关掉socket
         failed++;
         close(s);
         continue;
      }
      if (http10 == 0)
         if (shutdown(s, 1))
         {
            failed++;
            close(s);
            continue;
         }
      if (force == 0)
      {
         /* read all available data from socket */
         while (1)
         {
            if (timerexpired)
               break;
            i = read(s, buf, 1500);
            /* fprintf(stderr,"%d\n",i); */
            if (i < 0)
            {
               failed++;
               close(s);
               goto nexttry;
            }
            else if (i == 0)
               break;
            else
               bytes += i;
         }
      }
      if (close(s))
      {
         failed++;
         continue;
      }
//一次建立连接并读写成功，speed+1；
      speed++;
   }
}
```

# 修改webbench支持长连接

我们知道HTTP协议采用“请求-应答”模式，当使用普通模式，即非KeepAlive模式时，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP协议为无连接的协议）；当使用Keep-Alive模式（又称持久连接、连接重用）时，Keep-Alive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接。

http 1.0中默认是关闭的，需要在http头加入"Connection: Keep-Alive"，才能启用Keep-Alive；http 1.1中默认启用Keep-Alive，如果加入"Connection: close "，才关闭。目前大部分浏览器都是用http1.1协议，也就是说默认都会发起Keep-Alive的连接请求了，所以是否能完成一个完整的Keep-Alive连接就看服务器设置情况。

**优点明显：避免了建立/释放连接的开销**

benchcore.c中增加：

- 在循环外创建socket，每次重复通过这个socket读写，而不在重新创建socket。
- 前面参数处理，`while((opt=getopt_long(argc,argv,"912Vfrt:p:c:?hk",long_options,&options_index))!=EOF` 部分的代码修改。

socket.c中增加

- SO_LINGER 设置为onoff=1，linger=1，进行优雅关闭。

## 核心修改:

原来代码

```c
if (http10 > 1)
      strcat(request, "Connection: close\r\n");
   /* add empty line at end */
   if (http10 > 0)
      strcat(request, "\r\n");
   // printf("Req=%s\n",request);
```

修改为

```c
if(http10>1)
    {
        if (!keep_alive)
            strcat(request,"Connection: close\r\n");
        else
            strcat(request,"Connection: Keep-Alive\r\n");
    }
```

```c
if (keep_alive)
    {
        while (timerexpired == 0 && (s = Socket(host, port)) == -1){ };
        //s = Socket(host, port);
    nexttry1:
        while (1)
        {
            if (timerexpired)
            {
                if (failed > 0)
                {
                    /* fprintf(stderr,"Correcting failed by signal\n"); */
                    failed--;
                }
                return;
            }

            if (s < 0)
            {
                failed++;
                continue;
            }
            if (rlen != write(s, req, rlen))
            {
                failed++;
                close(s);
                while (!timerexpired == 0 && (s = Socket(host, port)) == -1){};
                //s = Socket(host, port);
                continue;
            }
            if (force == 0)
            {
                /* read all available data from socket */
                while (1)
                {
                    if (timerexpired)
                        break;
                    i = read(s, buf, 1500);
                    /* fprintf(stderr,"%d\n",i); */
                    if (i < 0 && errno == EAGAIN)
                    {
                        usleep(200);
                        continue;
                    }
                    else if (i < 0)
                    {
                        failed++;
                        close(s);
                        //while ((s = Socket(host,port)) == -1);
                        goto nexttry1;
                    }
                    else if (i == 0)
                        break;
                    else
                        bytes += i;
                    // Supposed reveived bytes were less than 1500
                    //if (i < 1500)
                    break;
                }
            }
            speed++;
        }
    }
```

# 测试

```c
cwd@cwd:~/work/WebBench$ webbench -t 10 -c 1000 -2 --get -k  http://www.baidu.com/
Webbench - Simple Web Benchmark 1.5
Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.
Runing info: 1000 clients, running 10 sec.

Speed=67344 pages/min, 1410208 bytes/sec.
Requests: 11224 susceed, 0 failed.
cwd@cwd:~/work/WebBench$ webbench -t 10 -c 1000 -2 --get   http://www.baidu.com/
Webbench - Simple Web Benchmark 1.5
Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.
Runing info: 1000 clients, running 10 sec.

Speed=6030 pages/min, 1431723 bytes/sec.
Requests: 1005 susceed, 0 failed.
```

# 解决webbench运行时卡住的问题

# 问题描述：

使用webbench进行压力测试的时候，在baidu上一般来说正常。在自己的webserver上测试，有时会卡住而没有返回值。在修改为长连接的时候尤其会发生。

如下图

![Untitled](webbench%20408cdfc4ccd44247ad802140e4410cad/Untitled%201.png)

必须要ctrl+c终止。

github上也有这样的问题

[长连接问题 · Issue #31 · linyacool/WebServer](https://github.com/linyacool/WebServer/issues/31)

# 解决思路：

找到可能被卡住的地方

测试时修改了部分源码进行打印，看每次1000个子进程中被阻塞了的进程数判断。

1：代码健壮性增加：从`while (s = Socket(host, port) == -1){};`  修改为

 `while (!timerexpired == 0 && (s = Socket(host, port)) == -1){};` 避免反复创建socket一直失败陷入死循环而不理会时钟到期的回调函数早就把`timerexpired` 置为1，通知进程该结束了。

**测试，没有明显好转。**

2：修改socket函数，增加将创建的套接字使用fcntl设置fd为 `O_NONBLOCK` 非阻塞的代码。可以避免在read的时候阻塞，而服务器已经结束发送，从而无限等待下去。接着要对应修改前面的代码，当read返回-1，但是errno为EAGAIN的时候，不能计入fail，可能只是服务器响应慢了一点而以，设计为让他睡一下下，再继续。

```c
if (i < 0 && errno == EAGAIN)
{
    usleep(200);
    continue;
}
```

**测试，略有好转。**

3：在测试时使用top查看，发现存在大量僵尸进程，卡住的时候僵尸进程还是很多。找到问题，有很多子进程是exit关闭的，父进程在fscanf，无法处理。导致clien—没有执行，无法跳出这个while循环。那么解决办法也就很明显了，给父进程也设置一个计时器与回调函数，到时间了就跳出这个循环，而不是使用while(1)死循环。

```c
//测试时有时候webbench卡住，就应该是在这个while1循环里卡住的。
      while (1)
      {
/*
fscanf 函数原型为 int fscanf(FILE * stream, const char * format, [argument...]); 
其功能为根据数据格式(format)，从输入流(stream)中读入数据，
存储到argument中，遇到空格和换行时结束。
*/
         pid = fscanf(f, "%d %d %d", &i, &j, &k);
         if (pid < 2)
         {
            fprintf(stderr, "Some of our childrens died.\n");
            break;
         }
         speed += i;
         failed += j;
         bytes += k;
         /* fprintf(stderr,"*Knock* %d %d read=%d\n",speed,failed,pid); */
         if (--clients == 0)
            break;
      }
```

增加代码

```c
static void father_alarm_handler(__attribute__((unused)) int signal)
{
    time_up_hard = 1;
}

...
struct sigaction sa_father;
/* setup alarm signal handler */
sa_father.sa_handler = father_alarm_handler;
sa_father.sa_flags = 0;
if (sigaction(SIGALRM, &sa_father, NULL))
    exit(3);

alarm(benchtime + 5); // after benchtime,then exit
...
while (!time_up_hard)
        {
            pid = fscanf(f, "%d %d %d", &i, &j, &k);
...
```

测试，可以按时跳出。显示有很多的failed。（自己的玩具webserver还是不够好）

以上的修改增加了webbench的健壮性。但是需要注意的是，即使在改装webbench前，使用baidu测试并不会出现问题，说明自己的webserver还是有待完善。
