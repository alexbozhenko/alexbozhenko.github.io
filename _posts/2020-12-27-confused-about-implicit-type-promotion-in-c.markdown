---
layout: post
title:  "Confused about implicit type promotion in C"
date:   2020-12-27 13:53:02 -0800
categories: C
---
The following code compiles and prints the message.

{% highlight c %}
#include <stdio.h>

char test_char = 'a';
int test_int = 97;

int main(void)
{
    if (test_char == test_int)
    {
        printf("%s\n", "compared int and char with no issues");
    }
}
{% endhighlight %}

I compile it with:
```
/usr/bin/gcc -Wall -Wextra -Wconversion -Werror -Wfloat-equal -Wmissing-noreturn -Wmissing-prototypes -Wsequence-point -Wshadow -Wstrict-prototypes -Wunreachable-code -pedantic -std=c18 -ggdb3
```

However, no warning or errors are printed, even though I compare `int` to `char`.

Googling shows that this is called ["Implicit type promotion"](https://stackoverflow.com/questions/46073295/implicit-type-promotion-rules)
and looks like compiler can not be configured to catch this behavior, since it is actually a language feature.

Some static analysis tools can potentially detect this. `Clang` static analyzer has a check called
[Loss of sign/precision in implicit conversions](https://clang.llvm.org/docs/analyzer/checkers.html#alpha-core-conversion-c-c-objc),
which is the closest thing to behavior I want.
One can run it by prepending the compilation command with `scan-build`. But still, it does not catch ANY instance of conversion, only 
those were loss of precision is happening.

{% highlight bash %}
$ scan-build -enable-checker alpha.core.Conversion /usr/bin/gcc -Wall -Wextra -Wconversion -Werror -Wfloat-equal -Wmissing-noreturn -Wmissing-prototypes -Wsequence-point -Wshadow -Wstrict-prototypes -Wunreachable-code -pedantic -std=c18 -ggdb3 test.c -o test.exe
scan-build: Using '/usr/bin/clang-11' for static analysis
scan-build: Analysis run complete.
scan-build: Removing directory '/tmp/scan-build-2020-12-28-174113-109891-1' because it contains no reports.
scan-build: No bugs found.
{% endhighlight %}

Coming from `OCaml`, it is shocking that this code works, and I need to make an effort to make it stop compiling.
I just had a bug in my code because I wrote 

```if (i == '\n')```

instead of 

```if (line[i] == '\n')```

And I would like to find out a way to not waste any time on bugs like this :)

[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/INT02-C.+Understand+integer+conversion+rules) also talks about this, and has list of paid static analysis tools that can catch the behavior to some extent.

UPD:
Also, when you have wrong specifier for `printf`, it also will convert the value for you:

{% highlight c %}
printf("uint range:\t[0;%d]\n", UINT_MAX);    // 2^32 = 4 bytes
printf("ulong range:\t[0;%ld]\n", ULONG_MAX); // 2^64 = 8 bytes
{% endhighlight %}

Prints:
```
uint range:     [0;-1]
ulong range:    [0;-1]
```

To get the correct values, the specifiers should be `u` and `lu` instead.
