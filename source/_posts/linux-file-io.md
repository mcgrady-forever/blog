---
title: 比较read/write & fread/fwrite
date: 2017-12-07 18:21:43
tags:
---

UNIX环境下的C对二进制流文件的读写有两套：
1. fopen,fread,fwrite,fprintf
2. open, read, write

##区别
1. fopen 系列是标准的C库函数；open系列是 POSIX 定义的，是UNIX系统里的system call。
也就是说，fopen系列更具有可移植性；而open系列只能用在 POSIX 的操作系统上。
2. 使用fopen 系列函数时要定义一个指代文件的对象，被称为“文件句柄”（file handler），是一个结构体；而open系列使用的是一个被称为“文件描述符” （file descriptor）的int型整数。
3. fopen 系列是级别较高的I/O，读写时使用缓冲；而open系列相对低层，更接近操作系统，读写时没有缓冲。由于能更多地与操作系统打交道，open系列可以访问更改一些fopen系列无法访问的信息，如查看文件的读写权限。这些额外的功能通常因系统而异。
4. 使用fopen系列函数需要"#include <sdtio.h>"；使用open系列函数需要"#include <fcntl.h>" ，链接时要之用libc（-lc）

##在开源项目中的运用
redis中写日志使用的是第一套,下面是redis的源码
```
void serverLogRaw(int level, const char *msg) {
    const int syslogLevelMap[] = { LOG_DEBUG, LOG_INFO, LOG_NOTICE, LOG_WARNING };
    const char *c = ".-*#";
    FILE *fp;
    char buf[64];
    int rawmode = (level & LL_RAW);
    int log_to_stdout = server.logfile[0] == '\0';

    level &= 0xff; /* clear flags */
    if (level < server.verbosity) return;

    fp = log_to_stdout ? stdout : fopen(server.logfile,"a");
    if (!fp) return;

    if (rawmode) {
        fprintf(fp,"%s",msg);
    } else {
        int off;
        struct timeval tv;
        int role_char;
        pid_t pid = getpid();

        gettimeofday(&tv,NULL);
        off = strftime(buf,sizeof(buf),"%d %b %H:%M:%S.",localtime(&tv.tv_sec));
        snprintf(buf+off,sizeof(buf)-off,"%03d",(int)tv.tv_usec/1000);
        if (server.sentinel_mode) {
            role_char = 'X'; /* Sentinel. */
        } else if (pid != server.pid) {
            role_char = 'C'; /* RDB / AOF writing child. */
        } else {
            role_char = (server.masterhost ? 'S':'M'); /* Slave or Master. */
        }
        fprintf(fp,"%d:%c %s %c %s\n",
            (int)getpid(),role_char, buf,c[level],msg);
    }
    fflush(fp);

    if (!log_to_stdout) fclose(fp);
    if (server.syslog_enabled) syslog(syslogLevelMap[level], "%s", msg);
}
```

google开源的日志系统glog写文件操作也采用第一套io函数，在函数LogFileObject::Write