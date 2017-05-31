---
title: C 语言的字符串
layout: post
comments: true
language: chinese
category: [linux,program]
keywords: linux,c,cpp,c++,program,string
description: 简单介绍下 C 语言中与字符串操作相关的函数。
---

简单介绍下 C 语言中与字符串操作相关的函数。

<!-- more -->

## 简介

C 语言中通过双引号表示字符串。

{% highlight c %}
#include <stdio.h>
int main (int argc, char **argv)
{
  char *ptr = "Hello world";
  puts(ptr);
  return 0;
}
{% endhighlight %}

可以理解为，双引号做了 3 件事：

1. 在只读常量区申请空间存放了字符串；
2. 在字符串尾添加了'\0'；
3. 返回字符串开始地址。

注意，可以把字符串赋值给字符指针 ```char *ptr```，而不能把字符串赋值给一个字符数组。

{% highlight c %}
char a[10] = "hello"; // 支持初始化方式
a = "hello";          // 此时错误
{% endhighlight %}

```"hello"``` 返回的是一个地址，而 a 虽然也有地址，但是这与指针是不一样的，指针的值是地址，而数组的值虽然也是地址，但是却是一个常量，所以不能给常量赋值；此时需要通过 ```strncpy()``` 复制。

另外，数组会分配存储空间，而 ```char *``` 实际上只是一个指针，当尝试复制到只读数据区时，那么此时就会导致 ```"Segmentation fault"``` 。

{% highlight c %}
#include <stdio.h>
#include <string.h>
int main (int argc, char **argv)
{
// 如下代码会出现"Segmentation fault"
//  char *ptr = "Hello world";
//  memcpy(ptr, "Hi foobar", strlen(ptr));

  char ptr[] = "Hello world";
  memcpy(ptr,  "Hey foobar!!!!!!!", sizeof(ptr));
  ptr[sizeof(ptr)-1] = '\0';
  puts (ptr);
}
{% endhighlight %}

### 检查宏

对于字符串操作，在 gcc 中，可以通过 ```#define _FORTIFY_SOURCE 2``` 宏定义，对一些常见内存以及字符串处理函数 (memcpy, strcpy, stpcpy, gets 等) 的安全检查，详细可以查看 ```man 7 feature_test_macros``` 。

下面以字符串拼接函数为例。

{% highlight c %}
#inclue <string.h>
char * strncat(char *dest, const char *src, size_t n);
{% endhighlight %}

将 src 中开始的 n 个字符复制到 dest 的结尾，也就是将 dest 字符串最后的 ```'\0'``` 覆盖掉，字符追加完成后，再追加 ```'\0'```；所以需要保证 dest 最小为 ```strlen(dest)+n+1``` 。

<!--
BAD: strncat(buffer,charptr,sizeof(buffer))
GOOD: strncat(buffer,charptr,sizeof(buffer)-strlen(buffer)-1)
-->

{% highlight c %}
#include <stdio.h>
#include <string.h>

int main(){
  char site[100] = "http://www.douniwai.com";
  char uri[300] = "/c/strncatiii";
  strncat(site, uri, 1000);  // BAD, 1000远远超过path的长度，那么可能会导致segment fault
  strncat(site, uri, sizeof(site)-strlen(site)-1); // GOOD
  puts(site);
  return  0;
}
{% endhighlight %}

可以通过 ```gcc -Wall -D_FORTIFY_SOURCE=2 -O2 main.c``` 命令编译测试。

















## 内存申请

{% highlight c %}
#include <stdlib.h>
void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
{% endhighlight %}

malloc() 申请 size 字节大小大内存，内存空间未初始化，如果 size 是 0 可能会返回 NULL 或者一个后续可以 free 的指针；分配内存失败则返回 NULL；calloc() 为申请数组空间，会自动初始化为 0 。

realloc() 的功能比较复杂，可以实现内存分配、释放，会将 ptr 所指向的内存块的大小修改为 size，并将新的内存指针返回，注意，不会保证返回的指针不变，会保证内存地址连续。注意，使用 malloc() 替换 realloc() 。

* 如果 ptr 和 size 均不为 0 时，会调整内存大小，返回新的开始指针，并保证源 size 内容不变；
* 如果 ptr 为 NULL 而 size 不为 0 ，那么等同于 malloc(size)；
* 如果 size = 0，那么相当于调用 free(ptr)。

一般来说，一个函数只提供一个功能，如上的函数却赋予了好几个功能，这并不是良好的函数设计，估计也是为了兼容性，才容忍这个函数一直在 C 库中。

另外，根据 C 的标准 [ISO-IEC 9899 7.20.3.2/2](http://www.open-std.org/JTC1/SC22/wg14/www/docs/n1124.pdf)：

{% highlight text %}
void free(void *ptr);
  If ptr is a null pointer, no action occurs.
{% endhighlight %}

所以，在执行 free() 时，不需要检查是否为 NULL，因此可以实现一个 sfree() 宏。

{% highlight c %}
#define sfree(m) do { \
    free(m);          \
    m = NULL;         \
} while(0);
{% endhighlight %}


### 注意事项

realloc()有可能操作失败，返回NULL，所以不要把它的返回值直接赋值给原来的指针变量，以免原值丢失：

{% highlight c %}
// 错误处理，返回失败时，原指针丢失，导致内存泄露
void *ptr = realloc(ptr, new_size);
if (!ptr) {
}
// 正确使用方法
void *new_ptr = realloc(ptr, new_size);
if (!new_ptr) {
}
ptr = new_ptr
{% endhighlight %}

一般 malloc(0) 和 calloc(0) 是合法的语句，会返还一个合法的指针，且该指针可以通过 free() 去释放；而对于 realloc() 则有不同的行为，如果为 0 ，那么最后再次 free() 时，就会导致重复释放。

### 示例

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
  int *p = NULL;
  p = (int *) malloc(sizeof(int));
  if (NULL == p) {
    printf("Can't get memory!\n");
    exit(EXIT_FAILURE);
  }
  memset(p, 0, sizeof(int));
  *p = 2;
  printf("%d\n", *p);
  free(p);

  #define SIZE 5
  int i = 0;
  p = (int *) calloc(SIZE, sizeof(int));
  if (NULL == p) {
    printf("Can't get memory!\n");
    exit(EXIT_FAILURE);
  }
  for (i = 0; i < SIZE; i++) {
    p[i] = i;
  }
  for (i = 0; i < SIZE; i++) {
    printf("p[%d]=%d\n", i, p[i]);
  }
  return 0;
}
{% endhighlight %}




## 字符串格式化

{% highlight c %}
#include <stdio.h>
int printf(const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
int sprintf(char *str, const char *format, ...);
int snprintf(char *str, size_t size, const char *format, ...);
{% endhighlight %}

除了上述一类的格式化函数之外，还有一类是上述函数添加了 v 开头的函数，如 ```vprintf()```、```vfprintf()``` 等；区别是后或者可以用户自定义打印函数，示例如下：

{% highlight c %}
int error(char *fmt, ...)
{
    int result;
    va_list args;
    va_start(args, fmt);
    fputs("Error: ", stderr);
    result = vfprintf(stderr, fmt, args);
    va_end(args);
    return result;
}
{% endhighlight %}

其中 ```printf()``` 和 ```fprintf()``` 输出到缓冲区，所以不需要考虑溢出，对于 ```sprintf()``` 和 ```snprintf()``` 建议使用后者；后者最多从源串中拷贝 n-1 个字符到目标串中，然后再在字符串后面添加 ```'\0'``` 。

若成功则返回存入数组的字符数(不含```'\0'```)；如果空间不足，则返回 "如果有足够空间存储，所应能输出的字符数(不含'\0')"，所以可以通过判断返回值是否大于 n 决定是否有溢出；若编码出错则返回负值。

如下是一个简单的示例。

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>

int main( int argc, char **argv)
{
  char buffer[255];
  int offset = 0, i;

  for ( i = 0; i < 10; i++) {
    offset += snprintf(buffer + offset, 255 - offset, ":%x\0", i);
  }
  printf ("buffer: %s\n", buffer);
  return 0;
}
{% endhighlight %}



### 示例

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>

int main()
{
  char str[10];
  int ret;

  // 超过10个字符，返回应该使用的strlen()，也就是12
  ret = snprintf(str, sizeof(str), "Hey %s !", "foobar");
  if (ret < 0) {
    perror("snprintf");
    exit(EXIT_FAILURE);
  } else if (ret >= sizeof(str)) {
    fprintf(stderr, "no enough buffer\n");
    printf("%s, ret %d\n", str, ret);
  } else {
    printf("%s, ret %d\n", str, ret);
  }

  return 0;
}
{% endhighlight %}


## 字符串整形转换

将字符串转换为长整型数，会扫描参数 nptr 字符串，跳过前面的空白字符 (如空格、tab等)，直到遇上数字或正负符号才开始做转换，然后再遇到非数字或字符串结束符(```'\0'```)结束转换，并将结果返回。

{% highlight text %}
#include <stdlib.h>
long int strtol (const char *nptr, char **endptr, int base);
long long int strtoll(const char *nptr, char **endptr, int base);

参数：
  str    : 要转换的字符串；
  endstr : 第一个不能转换的字符的指针，非NULL传回不符合条件而终止的字符串指针；
  base   : 字符串str所采用的进制，范围从2~36，也可以为0(默认为10进制；遇到0x/0X前置采用16进制；遇到0前置采用8进制) 。
返回值：
  转换后的长整型数；如果不能转换或者nptr为空字符串，返回 0(0L)；
  如果转换得到的值超出long int所能表示的范围，函数将返回LONG_MAX或LONG_MIN (在limits.h中定义)，并将errno设置为ERANGE。
{% endhighlight %}

ANSI C 规范定义了 ```stof()```、```atoi()```、```atol()```、```strtod()```、```strtol()```、```strtoul()``` 共 6 个可以将字符串转换为数字的函数；在 C99/C++11 规范中又新增了 5 个函数，分别是 ```atoll()```、```strtof()```、```strtold()```、```strtoll()```、```strtoull()```。

```atoi()``` 和 ```strtol()``` 的区别在于，前者无法判断转换是否成功，后者可以通过返回指针以及 errno 判断是否转换成功。

<!-- 如何通过宏定义判断是否支持后面的函数。-->

### 示例

{% highlight c %}
#include <stdio.h>      /* printf */
#include <stdlib.h>     /* strtol */
#include <errno.h>      /* errno */
#include <limits.h>     /* LONG_MAX, LONG_MIN */

int main (int argc, char **argv)
{
  char *numbers = "2001 60c0c0 0351 -1101110100110100100000 0x6fffff";
  char *endptr;
  long int li1, li2, li3, li4, li5;
  errno = 0;
  li1 = strtol (numbers, &endptr, 10);
  if ((errno == ERANGE && (li1 == LONG_MAX || li1 == LONG_MIN))
                || (errno != 0 && li1 == 0)) {
      perror("strtol");
      exit(EXIT_FAILURE);
  }
  if (endptr == numbers) { // || *pEend != '\0'
    fprintf(stderr, "No digits were found\n");
    exit(EXIT_FAILURE);
  }
  if (*endptr != '\0')
    printf("Further characters after number: %s\n", endptr);
  li2 = strtol (endptr, &endptr, 16);
  li3 = strtol (endptr, &endptr, 8);
  li4 = strtol (endptr, &endptr, 2);
  li5 = strtol (endptr, NULL, 0);
  printf ("The decimal equivalents are: %ld, 0x%lX, 0%lo, %ld and 0x%lX.\n", li1, li2, li3, li4, li5);

  return 0;
}
{% endhighlight %}


## 字符串复制

{% highlight c %}
#include <string.h>
char *strcpy(char *dest, const char *src);
char *strncpy(char *dest, const char *src, size_t n);
{% endhighlight %}

前者会将 src 复制到 dest ，包括 ```'\0'``` 字符，需要保证 dest 有足够的空间，否则可能会发生溢出。而后者最多复制 n 个字符串到 dest ，注意，如果这 n 个字符中没有 ```'\0'``` 那么需要手动添加。

返回字符串 dest 。

### strdup()

该函数会通过 ```malloc()``` 申请内存，然后将字符串复制过去，所以需要调用者手动释放内存。

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
  char *src ="This is the jibo";
  char *dest;
  dest = strdup(src);
  printf("the dest %s\n",dest);
  free(dest);

  return 0;
}
{% endhighlight %}

如果不通过 ```free()``` 释放内存，那么会导致内存泄露，可以通过 valgrind 命令检查，如下：

{% highlight text %}
$ valgrind --tool=memcheck --leak-check=yes ./test
{% endhighlight %}

### memcpy VS. memmove

{% highlight c %}
#include <stdio.h>
#include <string.h>
int main ()
{
  char str[] = "memmove can be very useful......";
  memmove (str+20, str+15, 11);
  puts (str);
  return 0;
}
{% endhighlight %}

注意，即使替换为 ```memcpy()``` 上述的代码仍有可能是正确的，这主要是由于编译器的优化所致，详细可以反汇编查看。

{% highlight c %}
#include <string.h>
void *memcpy(void *dest, const void *src, size_t n);
void *memmove(void *dest, const void *src, size_t n);
{% endhighlight %}

两者都是将 N 个字节的源内存地址的内容拷贝到目标内存地址中，需要保证 dest 有足够的空间；区别是后者可以有效处理重叠区域<!--，首先会将 src 的内容复制到临时缓存区，然后再复制到 dest 中-->，其处理措施如下：

* 当源内存的首地址等于目标内存的首地址时，不进行任何拷贝；
* 当源内存的首地址大于目标内存的首地址时，实行正向拷贝；
* 当源内存的首地址小于目标内存的首地址时，实行反向拷贝；

更详细的场景可以分为如下：

{% highlight text %}
内存低端 <-----s-----> <-----d-----> 内存高端 start at end of s
内存低端 <-----s--<==>--d----->      内存高端 start at end of s
内存低端 <-----sd----->              内存高端 do nothing
内存低端 <-----d--<==>--s----->      内存高端 start at beginning of s
内存低端 <-----d-----> <-----s-----> 内存高端 start at beginning of s
{% endhighlight %}

如下是两者的实现代码：

{% highlight c %}
void* memcpy(void* dest, const void* src, size_t n)
{
  char*        d = (char*) dest;
  const char*  s = (const char*) src;
  while(n--)
    *d++ = *s++;
  return dest;
}

void* memmove(void* dest, const void* src, size_t n)
{
  char*        d = (char*) dest;
  const char*  s = (const char*) src;

  if (s>d) {        // start at beginning of s
    while (n--)
      *d++ = *s++;
  } else if (s<d) { // start at end of s
    d = d+n-1;
    s = s+n-1;
    while (n--)
      *d-- = *s--;
  }
  return dest;
}
{% endhighlight %}

与之相关的还有 ```memccpy()```、```mempcpy()``` 以及 ```bcopy()``` 函数：

* ```memccpy()``` 将 src 指向内存的前 n 个字节复制到 dest ，期间在遇到某个特定值 (int c) 就立即停止复制。
* ```mempcpy()``` 类似于 ```memcpy()``` ，不同的是该函数返回复制到最后地址的下一个地址，而非返回 dest，通常用于流式的复制。


{% highlight c %}
{% endhighlight %}