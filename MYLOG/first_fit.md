# First-fit paradigm
> LongChampion, 12/03/2020

**GLIBC** use *first-fit algorithm* to select free chunk to return.  
It will iterate each free chunk (in some pre-defined rules) and the first chunk satisfied all condition is returned to user.  
I have made an amazing example:
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    void *a = malloc(0x400);
    void *b = malloc(0x400);

    strcpy(a, "AAAA");
    strcpy(b, "BBBB");

    printf("chunk a contains %s\n", a);
    printf("chunk b contains %s\n", b);

    free(a);

    size_t size;
    printf("Enter the size: ");
    scanf("%lu", &size);

    printf("%lu\n", size);
    void *your = malloc(size);
    strcpy(your, "YOUR");

    printf("your chunk contains %s\n", your);
    printf("chunk a contains %s\n", a);
    printf("chunk b contains %s\n", b);

    free(your);
    free(b);
}
```
and there is the output:
```
chunk a contains AAAA
chunk b contains BBBB
Enter the size: 1000
1000
your chunk contains YOUR
chunk a contains 1000

chunk b contains BBBB
```
hmm, everything look ok, but why chunk `a` contains "1000"???  
If you use `GDB` to trace into `scanf`, you will see it call `malloc(0x400)` somewhere. Then by *first-fit paradigm*, **GLIBC** return chunk `a` to serve `scanf`. As a result, chunk `a` contains our input ("1000\n") and this is the reason why we have the output above.
