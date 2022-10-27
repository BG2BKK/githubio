+++
date = '2016-05-18T15:56:47+08:00'
draft = true
title = 'signal信号编程时的SA_RESTART用法'

+++

SA_RESTART
----------------

```cpp
// from http://www.win.tue.nl/~aeb/linux/lk/lk-4.html#ss4.5
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>

int got_interrupt;

void intrup(int dummy) {
        got_interrupt = 1;
        char *msg = "got signal\n";
        write(1, msg, strlen(msg));
}

void die(char *s) {
        printf("%s\n", s);
        exit(1);
}

int main() {
        struct sigaction sa;
        int n;
        char c;

        sa.sa_handler = intrup;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        if (sigaction(SIGINT, &sa, NULL))
                die("sigaction-SIGINT");
        sa.sa_flags = SA_RESTART;
        if (sigaction(SIGQUIT, &sa, NULL))
                die("sigaction-SIGQUIT");

        got_interrupt = 0;
        n = read(0, &c, 1);
        if (n == -1 && errno == EINTR)
                printf(">>> read call was interrupted\n");
        else if (got_interrupt)
                printf(">>> read call was restarted\n");

        printf("Read character: %c\n", c);

        return 0;
}

```bash
$ ./sa_restart.o 
^cgot signal		-- 这里按下Crtl+C，发送SIGINT信号
>>> read call was interrupted
read character: 
```

TODO LIST
-------------

* O_CLOEXEC
* [EPOLLEXCLUSIVE](https://lkml.org/lkml/2016/3/10/515)

$ ./sa_restart.o 
^cgot signal
>>> read call was interrupted
read character: 

