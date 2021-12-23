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

