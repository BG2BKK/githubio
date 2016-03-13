+++
date = "2016-02-27T09:49:41+08:00"
draft = false
title = "pipe在内核中的实现"

+++

## pipe在linux内核中的实现

在之前关于linux shell多线程并发数控制的[博文](https://bg2bkk.github.io/post/shell%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%AE%9E%E8%B7%B5/)中，我们使用了fifo作为token池，通过读写fifo实现token分发控制，进而实现了控制线程数的目的。 
我对pipe这个*** *NIX ***系统中最常用的组件（|）产生了兴趣 

- fifo和pipe是什么关系？
- fifo或者pipe的使用方法？
- pipe在linux kernel中的实现是怎样的？
- fifo或者pipe的容量有多大，可以配置吗？

先说结论吧

- fifo和pipe的区别
    * pipe是匿名管道，没有名字，只能用于两个拥有pipe读写两端fd的进程通信；
    * fifo在文件系统中有自己的名称，操作fifo与操作普通文件几无差别，可以用于两个没有关系的进程间通信
- fifo和pipe在kernel层面上都实现在fs/pipe.c中，所以本质上二者是一个东西。
- pipe作为linux文件系统的一部分，与epoll一样，都是在向kernel注册了自己的文件系统，可以使用VFS提供的通用接口，比如open、read和write等操作
- pipe的容量不是无限大的，早期linux版本（kernel-2.4）中pipe容量只能是4KB大小，新版本可以在运行时根据需要扩大到64KB

本文主要基于linux-2.4.20内核中的pipe实现进行分析，理由是该版本的pipe实现与新版本kernel并没有太大差别，但是代码可读性要强很多，可以快速了解pipe的实际实现；从2.4.20内核中对pipe的架构有整体了解后，再阅读新版本(4.4.1)中的新feature，会比较顺遂。

### pipe在fs/pipe.c中一些函数
    - new_inode()
    - register_filesystem()
    - get_empty_file
    - get_unused_fd()
    - do_pipe作为pipe系统调用函数，在/arch/i386/sys_i386.c中定义
    - pipe的module_init在initcall中调用，但是pipe.c是编译在fs.o中的，他的module_init是如何调用进去的，需要进一步查找
    - struct dentry在include/linux/dcache.h中定义

### 具体实现

    * pipe文件系统初始化
        * 注册pipefs
            * register_filesystems
    
    * pipe系统调用
        * pipe调用do_pipe
        * do_pipe()
            * f1&f2 get_empty_filep分配filep数据结构
            * inode = get_pipe_inode()从pipe文件系统获得inode
                * new_inode()
                * pipe_new()新建pipe
                    * __get_free_pages(GFP_USER)为该pipe分配一页内存（4KB）
                    * inode->i_pipe = kmalloc(sizeof(struct pipe_inde_info), GFP_KERNEL)分配pipe信息结构
            * i&j = get_unused_fd()获取两个fd
            * dentry = d_alloc()从pipefs分配dentry
            * d_add(dentry, inode)将inode插入到dentry中
            * 将f1设置成O_RDONLY，将f2设置成O_WRONLY
            * 进程的files列表中，files[i] = f1, files[j] = f2
    
    * 实现函数
        * pipe
            * pipe_read
            * pipe_write


* tips
    * pipe不允许使用seek
    * 低版本linux-2.4.20在pipe写的时候是固定大小，而高版本的是会按需分配直至64KB的。
    * 高版本kernel内核中sysctl的配置参数***fs.pipe-max-size*** 可以设置固定的pipe大小。但是也不能超过64KB大小，即使配置数据大于这个数字，pipe大小也会限制在64KB。

    * [测试代码](http://unix.stackexchange.com/questions/11946/how-big-is-the-pipe-buffer)

```bash
#!/bin/bash
test $# -ge 1 || { echo "usage: $0 write-size [wait-time]"; exit 1; }
test $# -ge 2 || set -- "$@" 1
bytes_written=$(
{
    exec 3>&1
    {
        perl -e '
            $size = $ARGV[0];
            $block = q(a) x $size;
            $num_written = 0;
            sub report { print STDERR $num_written * $size, qq(\n); }
            report; while (defined syswrite STDOUT, $block) {
                $num_written++; report;
            }
        ' "$1" 2>&3
    } | (sleep "$2"; exec 0<&-);
} | tail -1
)
printf "write size: %10d; bytes successfully before error: %d\n" \
    "$1" "$bytes_written"

```
    * 测试结果

```
huang@ThinkPad-X220:~/workspace/cpp$ /bin/bash -c 'for p in {0..18}; do ./pipe.sh $((2 ** $p)) 0.5; done'
write size:          1; bytes successfully before error: 65536
write size:          2; bytes successfully before error: 65536
write size:          4; bytes successfully before error: 65536
write size:          8; bytes successfully before error: 65536
write size:         16; bytes successfully before error: 65536
write size:         32; bytes successfully before error: 65536
write size:         64; bytes successfully before error: 65536
write size:        128; bytes successfully before error: 65536
write size:        256; bytes successfully before error: 65536
write size:        512; bytes successfully before error: 65536
write size:       1024; bytes successfully before error: 65536
write size:       2048; bytes successfully before error: 65536
write size:       4096; bytes successfully before error: 65536
write size:       8192; bytes successfully before error: 65536
write size:      16384; bytes successfully before error: 65536
write size:      32768; bytes successfully before error: 65536
write size:      65536; bytes successfully before error: 65536
write size:     131072; bytes successfully before error: 0
write size:     262144; bytes successfully before error: 0

```
    * 内核中64KB大小的限制在哪里设置的？(TO DO)
        * 只有在高版本的pipe实现中才有64KB大小，低版本都是4KB的。
        * ulimit -a 的结果中，"pipe size (512 bytes, -p) 8"，表示一个pipe拥有8个512KB的buffer，总共是4KB
        * 在include/linux/fs_pipe_i.h中，#define PIPE_DEF_BUFFERS   16, 这里是[按buffer的数量分配的](http://home.gna.org/pysfst/tests/pipe-limit.html)。
        * 在fs/pipe.c中，pipe_write和pipe_read是在运行时按页大小分配的
        * sysctl中fs.max_pipe_size的设置，fs.pipe-max-size = 1048576，又会起什么作用


* 函数分析
    * init_pipe_fs向文件系统注册pipe组件

```cpp
static int __init init_pipe_fs(void)
{
	int err = register_filesystem(&pipe_fs_type);
	if (!err) {
		pipe_mnt = kern_mount(&pipe_fs_type);
		err = PTR_ERR(pipe_mnt);
		if (IS_ERR(pipe_mnt))
			unregister_filesystem(&pipe_fs_type);
		else
			err = 0;
	}
	return err;
}
```

```cpp
static DECLARE_FSTYPE(pipe_fs_type, "pipefs", pipefs_read_super, FS_NOMOUNT);
```

```cpp
#define DECLARE_FSTYPE(var,type,read,flags) \
struct file_system_type var = { \
	name:		type, \
	read_super:	read, \
	fs_flags:	flags, \
	owner:		THIS_MODULE, \
}
```

    * pipe源码（2.4的实现中实在没什么可讲的，比较有价值的是pipe_write和pipe_read中处理缓冲队列源码可以参考）

```cpp
/*
 * We use a start+len construction, which provides full use of the 
 * allocated memory.
 * -- Florian Coosmann (FGC)
 * 
 * Reads with count = 0 should always return 0.
 * -- Julian Bradfield 1999-06-07.
 */

/* Drop the inode semaphore and wait for a pipe event, atomically */
void pipe_wait(struct inode * inode)
{
	DECLARE_WAITQUEUE(wait, current);
	current->state = TASK_INTERRUPTIBLE;
	add_wait_queue(PIPE_WAIT(*inode), &wait);
	up(PIPE_SEM(*inode));
	schedule();
	remove_wait_queue(PIPE_WAIT(*inode), &wait);
	current->state = TASK_RUNNING;
	down(PIPE_SEM(*inode));
}

static ssize_t
pipe_read(struct file *filp, char *buf, size_t count, loff_t *ppos)
{
	struct inode *inode = filp->f_dentry->d_inode;
	ssize_t size, read, ret;

	/* Seeks are not allowed on pipes.  */
	ret = -ESPIPE;
	read = 0;
	if (ppos != &filp->f_pos)
		goto out_nolock;

	/* Always return 0 on null read.  */
	ret = 0;
	if (count == 0)
		goto out_nolock;

	/* Get the pipe semaphore */
	ret = -ERESTARTSYS;
	if (down_interruptible(PIPE_SEM(*inode)))
		goto out_nolock;

	if (PIPE_EMPTY(*inode)) {
do_more_read:
		ret = 0;
		if (!PIPE_WRITERS(*inode))
			goto out;

		ret = -EAGAIN;
		if (filp->f_flags & O_NONBLOCK)
			goto out;

		for (;;) {
			PIPE_WAITING_READERS(*inode)++;
			pipe_wait(inode);
			PIPE_WAITING_READERS(*inode)--;
			ret = -ERESTARTSYS;
			if (signal_pending(current))
				goto out;
			ret = 0;
			if (!PIPE_EMPTY(*inode))
				break;
			if (!PIPE_WRITERS(*inode))
				goto out;
		}
	}

	/* Read what data is available.  */
	ret = -EFAULT;
	while (count > 0 && (size = PIPE_LEN(*inode))) {
		char *pipebuf = PIPE_BASE(*inode) + PIPE_START(*inode);
		ssize_t chars = PIPE_MAX_RCHUNK(*inode);

		if (chars > count)
			chars = count;
		if (chars > size)
			chars = size;

		if (copy_to_user(buf, pipebuf, chars))
			goto out;

		read += chars;
		PIPE_START(*inode) += chars;
		PIPE_START(*inode) &= (PIPE_SIZE - 1);
		PIPE_LEN(*inode) -= chars;
		count -= chars;
		buf += chars;
	}

	/* Cache behaviour optimization */
	if (!PIPE_LEN(*inode))
		PIPE_START(*inode) = 0;

	if (count && PIPE_WAITING_WRITERS(*inode) && !(filp->f_flags & O_NONBLOCK)) {
		/*
		 * We know that we are going to sleep: signal
		 * writers synchronously that there is more
		 * room.
		 */
		wake_up_interruptible_sync(PIPE_WAIT(*inode));
		if (!PIPE_EMPTY(*inode))
			BUG();
		goto do_more_read;
	}
	/* Signal writers asynchronously that there is more room.  */
	wake_up_interruptible(PIPE_WAIT(*inode));

	ret = read;
out:
	up(PIPE_SEM(*inode));
out_nolock:
	if (read)
		ret = read;

	UPDATE_ATIME(inode);
	return ret;
}

static ssize_t
pipe_write(struct file *filp, const char *buf, size_t count, loff_t *ppos)
{
	struct inode *inode = filp->f_dentry->d_inode;
	ssize_t free, written, ret;

	/* Seeks are not allowed on pipes.  */
	ret = -ESPIPE;
	written = 0;
	if (ppos != &filp->f_pos)
		goto out_nolock;

	/* Null write succeeds.  */
	ret = 0;
	if (count == 0)
		goto out_nolock;

	ret = -ERESTARTSYS;
	if (down_interruptible(PIPE_SEM(*inode)))
		goto out_nolock;

	/* No readers yields SIGPIPE.  */
	if (!PIPE_READERS(*inode))
		goto sigpipe;

	/* If count <= PIPE_BUF, we have to make it atomic.  */
	free = (count <= PIPE_BUF ? count : 1);

	/* Wait, or check for, available space.  */
	if (filp->f_flags & O_NONBLOCK) {
		ret = -EAGAIN;
		if (PIPE_FREE(*inode) < free)
			goto out;
	} else {
		while (PIPE_FREE(*inode) < free) {
			PIPE_WAITING_WRITERS(*inode)++;
			pipe_wait(inode);
			PIPE_WAITING_WRITERS(*inode)--;
			ret = -ERESTARTSYS;
			if (signal_pending(current))
				goto out;

			if (!PIPE_READERS(*inode))
				goto sigpipe;
		}
	}

	/* Copy into available space.  */
	ret = -EFAULT;
	while (count > 0) {
		int space;
		char *pipebuf = PIPE_BASE(*inode) + PIPE_END(*inode);
		ssize_t chars = PIPE_MAX_WCHUNK(*inode);

		if ((space = PIPE_FREE(*inode)) != 0) {
			if (chars > count)
				chars = count;
			if (chars > space)
				chars = space;

			if (copy_from_user(pipebuf, buf, chars))
				goto out;

			written += chars;
			PIPE_LEN(*inode) += chars;
			count -= chars;
			buf += chars;
			space = PIPE_FREE(*inode);
			continue;
		}

		ret = written;
		if (filp->f_flags & O_NONBLOCK)
			break;

		do {
			/*
			 * Synchronous wake-up: it knows that this process
			 * is going to give up this CPU, so it doesn't have
			 * to do idle reschedules.
			 */
			wake_up_interruptible_sync(PIPE_WAIT(*inode));
			PIPE_WAITING_WRITERS(*inode)++;
			pipe_wait(inode);
			PIPE_WAITING_WRITERS(*inode)--;
			if (signal_pending(current))
				goto out;
			if (!PIPE_READERS(*inode))
				goto sigpipe;
		} while (!PIPE_FREE(*inode));
		ret = -EFAULT;
	}

	/* Signal readers asynchronously that there is more data.  */
	wake_up_interruptible(PIPE_WAIT(*inode));

	inode->i_ctime = inode->i_mtime = CURRENT_TIME;
	mark_inode_dirty(inode);

out:
	up(PIPE_SEM(*inode));
out_nolock:
	if (written)
		ret = written;
	return ret;

sigpipe:
	if (written)
		goto out;
	up(PIPE_SEM(*inode));
	send_sig(SIGPIPE, current, 0);
	return -EPIPE;
}


/* No kernel lock held - fine */
static unsigned int
pipe_poll(struct file *filp, poll_table *wait)
{
	unsigned int mask;
	struct inode *inode = filp->f_dentry->d_inode;

	poll_wait(filp, PIPE_WAIT(*inode), wait);

	/* Reading only -- no need for acquiring the semaphore.  */
	mask = POLLIN | POLLRDNORM;
	if (PIPE_EMPTY(*inode))
		mask = POLLOUT | POLLWRNORM;
	if (!PIPE_WRITERS(*inode) && filp->f_version != PIPE_WCOUNTER(*inode))
		mask |= POLLHUP;
	if (!PIPE_READERS(*inode))
		mask |= POLLERR;

	return mask;
}

```

