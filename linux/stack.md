os是如何看待虚拟地址的？

mmap的底层实现原理是啥？

线程的栈、内核栈在os中到底是个什么东西，是个什么原理。

# 创建线程的时候，可以指定栈空间，原理是啥？

man clone详情如下：
https://man7.org/linux/man-pages/man2/clone.2.html

从这里可以看到，现在有三个系统调用与线程/进程的创建有关：
1. fork；
2. clone
3. clone3。

clone相比fork，可以更加精细化地创建task_struct(即PCB)；clone3相比clone，封装了参数到结构体，并额外新增了很多参数。（一个问题：fork调用clone作为实现的话，是如何指定函数指针参数的？）

以下是栈相关的结论：
1. clone的函数参数存在stack指针。但是没有指定大小，所以stack需要定位到栈顶上去，即申请空间之后要将指针设置为这段空间的末尾位置，然后传递给clone。由于没有指定大小，所以kernal也就无从知晓栈的大小，所以一旦使用超过了最初分配的，则会导致panic；
2. clone3则优化了栈参数的定义，使用了栈底指针，并同时提供了栈的大小参数。
3. 一旦clone的flag指定了CLONE_VM参数，即共享了虚拟地址空间，则栈指针参数必须指定，因为在同一个虚拟地址空间下，不能共享栈的使用；同理，如果没有指定这个参数，则无需指定栈空间，此时从虚拟地址角度，父子进程使用的栈空间地址是相同的。

# 如果没有指定栈空间，每个线程的栈空间是如何分配的呢？

栈的初始地址是在哪？从如下可以看出，每个进程的栈地址可能并不相同：
```
[serv@cn-hz-wl-test-devops-test00 ~]$ cat /proc/7501/maps | grep 'stack'
7ffcb447a000-7ffcb449b000 rw-p 00000000 00:00 0                          [stack]
[serv@cn-hz-wl-test-devops-test00 ~]$ cat /proc/7502/maps | grep 'stack'
7ffcb447a000-7ffcb449b000 rw-p 00000000 00:00 0                          [stack]
[serv@cn-hz-wl-test-devops-test00 ~]$ cat /proc/1491/maps | grep 'stack'
7ffe04ed3000-7ffe04ef4000 rw-p 00000000 00:00 0                          [stack]
[serv@cn-hz-wl-test-devops-test00 ~]$ cat /proc/1473/maps | grep 'stack'
7fff8bcfe000-7fff8bd1f000 rw-p 00000000 00:00 0                          [stack]
```

那exec一个线程的时候，会不会影响栈呢？

man execve 详情如下：
https://man7.org/linux/man-pages/man2/execve.2.html

execve会根据新的可执行文件，来生成新的栈空间。那到底如何生成的呢？

这里一个小技巧，新版linux查看系统调用的定义可以使用如下shell命令搜索：
```
fgrep 'SYSCALL_DEFINE' ./* -rin | grep 'execve'
```

可以发现execve的定义在fs目录下，这个是没有问题的，因为execve需要读取文件中可执行文件进行执行。

这篇文章对execve的源码解析的比较明白了：https://zhuanlan.zhihu.com/p/363510745

可以看到exec之后，线程的task_struct和栈都是重新赋值的，栈的赋值逻辑是
```
randomize_stack_top(STACK_TOP)
```
在常量STACK_TOP的基础上加一个随机数偏移。这个常量的定义与系统架构有关，不同的系统架构不太一样。

静态链接文件中不可能有段，因为没有必要存在，可执行程序中还可能存在。

# 创建线程时手动指定了栈指针，这个栈指针是怎么来的？

pthead_create的源码。glibc的源码下载地址：
https://mirror.csclub.uwaterloo.ca/gnu/libc/

pthread_create的源码在 nptl/pthread_create.c 中。
```
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)
{
    xxx
    struct pthread *pd = NULL;
    int err = allocate_stack (iattr, &pd, &stackaddr, &stacksize);
    int retval = 0;
    xxx
```
allocate_stack位于nptl/allocatestack.c中
```
/* Returns a usable stack for a new thread either by allocating a
   new stack or reusing a cached stack of sufficient size.
   ATTR must be non-NULL and point to a valid pthread_attr.
   PDP must be non-NULL.  */
static int
allocate_stack (const struct pthread_attr *attr, struct pthread **pdp,
		void **stack, size_t *stacksize)
    xxx
      /* Try to get a stack from the cache.  */
      reqsize = size;
      pd = get_cached_stack (&size, &mem);
      if (pd == NULL)
	{
	  /* If a guard page is required, avoid committing memory by first
	     allocate with PROT_NONE and then reserve with required permission
	     excluding the guard page.  */
	  mem = __mmap (NULL, size, (guardsize == 0) ? prot : PROT_NONE,
			MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
    xxx
```
可以看到，会尝试从缓存中获取，如果获取不到，则使用mmap来申请一块内存。其中mmap的flag的参数(man mmap)：
```
MAP_STACK (since Linux 2.6.27)
              Allocate the mapping at an address suitable for a process
              or thread stack.

              This flag is currently a no-op on Linux.  However, by
              employing this flag, applications can ensure that they
              transparently obtain support if the flag is implemented in
              the future.  Thus, it is used in the glibc threading
              implementation to allow for the fact that some
              architectures may (later) require special treatment for
              stack allocations.  A further reason to employ this flag
              is portability: MAP_STACK exists (and has an effect) on
              some other systems (e.g., some of the BSDs).
```
MAP_STACK：啥都没做。

# 线程栈的初始大小是多大

上面的allocate_stack函数中，mmap的size参数的由来：
```
  /* Get the stack size from the attribute if it is set.  Otherwise we
     use the default we determined at start time.  */
  if (attr->stacksize != 0)
    size = attr->stacksize;
  else
    {
      lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
      size = __default_pthread_attr.internal.stacksize;
      lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
    }
```
创建线程的时候，可以指定attr中的stacksize参数，如果没有指定，则取默认值（__default_pthread_attr.internal.stacksize），这个默认值的定义如下：
```
  /* Determine the default allowed stack size.  This is the size used
     in case the user does not specify one.  */
  struct rlimit limit;
  if (__getrlimit (RLIMIT_STACK, &limit) != 0
      || limit.rlim_cur == RLIM_INFINITY)
    /* The system limit is not usable.  Use an architecture-specific
       default.  */
    limit.rlim_cur = ARCH_STACK_DEFAULT_SIZE;
  else if (limit.rlim_cur < PTHREAD_STACK_MIN)
    /* The system limit is unusably small.
       Use the minimal size acceptable.  */
    limit.rlim_cur = PTHREAD_STACK_MIN;

  /* Make sure it meets the minimum size that allocate_stack
     (allocatestack.c) will demand, which depends on the page size.  */
  const uintptr_t pagesz = GLRO(dl_pagesize);
  const size_t minstack = (pagesz + __nptl_tls_static_size_for_stack ()
                           + MINIMAL_REST_STACK);
  if (limit.rlim_cur < minstack)
    limit.rlim_cur = minstack;

  /* Round the resource limit up to page size.  */
  limit.rlim_cur = ALIGN_UP (limit.rlim_cur, pagesz);
  __default_pthread_attr.internal.stacksize = limit.rlim_cur;
  __default_pthread_attr.internal.guardsize = GLRO (dl_pagesize);
```
其中：
```
struct rlimit
  {
    /* The current (soft) limit.  */
    rlim_t rlim_cur;
    /* The hard limit.  */
    rlim_t rlim_max;
  };
```
RLIMIT_STACK可以通过`ulimit -a`查到，一般是8M。总之，栈的大小，要么是ARCH_STACK_DEFAULT_SIZE(架构相关，如2m，4m)，要么是PTHREAD_STACK_MIN(131072)

# linux内核如何对栈自动扩容的
操作系统能识别一个虚拟内存区域是否为栈空间，如果识别为栈空间，会在缺页异常中自动为栈空间扩容，但不会超过RLIMIT_STACK的限制。

参考如下代码：
```
#include <iostream>
#include <unistd.h>

using namespace std;
void func() {
        char arr[8000000]; // < 8m
        for (int i = 0; i < 8000000; i++) {
                arr[i] = 97;
        }
        for (int i = 1000; i < 1010; i++) {
                cout << arr[i];
        }
}

void func1() {
        char arr[9000000]; // > 8m
        for (int i = 0; i < 8000000; i++) {
                arr[i] = 97;
        }
        for (int i = 1000; i < 1010; i++) {
                cout << arr[i];
        }
}

int main() {
        sleep(30);
        func();
        sleep(30);
        func1();
}
```
查看进程的maps文件可以看到：
```
[10.107.64.120:data-dev@dev:~]$ cat /proc/31326/maps | grep 'stack'
7ffc6cb5d000-7ffc6cb7e000 rw-p 00000000 00:00 0                          [stack]
[10.107.64.120:data-dev@dev:~]$ cat /proc/31326/maps | grep 'stack'
7ffc6c3da000-7ffc6cb7e000 rw-p 00000000 00:00 0                          [stack]
```
栈的虚拟空间被自动扩容，由之前的135168扩展到8011776。

os是如何识别是栈呢？缺页异常处理程序中（do_page_fault函数），有如下代码：
```
	vma = find_vma(mm, address);
	if (!vma)
		goto bad_area;
	if (vma->vm_start <= address)
		goto good_area;
	if (!(vma->vm_flags & VM_GROWSDOWN))
		goto bad_area;
	if (expand_stack(vma, address))
		goto bad_area;

	/* Ok, we have a good vm_area for this memory access, so
	   we can handle it.  */
 good_area:
    xxx
    /* If for any reason at all we couldn't handle the fault,
	make sure we exit gracefully rather than endlessly redo
	the fault.  */
	fault = handle_mm_fault(vma, address, flags, regs);
```
这里首先会根据address从PCB中找到对应的vma，根据vma的VM_GROWSDOWN来判断是否为栈区，只有栈才向下增长。如果发现是栈区，就会进入expand_stack中扩容，当然扩容会考虑RLIMIT_STACK限制，超过8m则会报错NOMEM。

expand_stack只会修改vma中栈的虚拟地址范围，不会实际分配物理地址。一旦expand_stack成功，会进入下面的good_area区域，之后进行物理页面的分配。

# golang中的栈的自动扩容是如何实现的呢？
自定义栈完全没有问题，关键在于如何自动扩容、自动缩容呢？何时检查是否需要自动扩容缩容呢？