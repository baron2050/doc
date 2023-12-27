# C常用函数

## strncasecmp()函数

头文件：#include <string.h>

定义函数：int strncasecmp(const char *s1, const char *s2, size_t n);

函数说明：strncasecmp()用来比较参数s1 和s2 字符串前n个字符，比较时会自动忽略大小写的差异。

返回值：若参数s1 和s2 字符串相同则返回0。s1 若大于s2 则返回大于0 的值，s1 若小于s2 则返回小于0 的值。

范例

```c
#include <string.h>
main(){    
    char *a = "aBcDeF";    
    char *b = "AbCdEf";    
    if(!strncasecmp(a, b, 3))    
        printf("%s =%s\n", a, b);
}
```


执行结果：
aBcDef=AbCdEf

## strchr()

C 库函数 **char \*strchr(const char \*str, int c)** 在参数 **str** 所指向的字符串中搜索第一次出现字符 **c**（一个无符号字符）的位置。

## 声明

下面是 strchr() 函数的声明。

```
char *strchr(const char *str, int c)
```

## 参数

- **str** -- 要被检索的 C 字符串。

- **c** -- 在 str 中要搜索的字符。

  ```c
  #include <stdio.h>
  #include <string.h>
  
  int main ()
  {
     const char str[] = "http://www.runoob.com";
     const char ch = '.';
     char *ret;
  
     ret = strchr(str, ch);
  
     printf("|%c| 之后的字符串是 - |%s|\n", ch, ret);
     
     return(0);
  }
  ```


## zstr 函数⽤于判断空字符串

zstr 函数⽤于判断空字符串

## atoi () 函数用来将字符串转换成整数 (int)

atoi () 函数用来将字符串转换成整数 (int)