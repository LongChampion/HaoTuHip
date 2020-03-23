# Fastbin duplicate into STACK
> LongChampion, 12/03/2020

> Please read `fastbin_dup.md` to know how to duplicate the chunk in fastbin linked list.

My code:
```c
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    struct Chunk Fake;
    void *a = malloc(100);
    void *b = mallloc(100);

    free(a), free(b), free(a);

    void *one = malloc(100);
    void *two = malloc(100);

    one = &Fake;
    Fake.size = 112;

    void *three = malloc(100);
    void *STACK = malloc(100);
}
```
The ideal is simple: we use *double free* to duplicate chunk `a`, then call `malloc` to receive it again. When we modify content of `one`, we also modify content of `three` (although which haven't allocated yet). The stage of fastbin before *hijacking* is:
> HEAD -> a -> TAIL

After *hijacking*:
> HEAD -> a -> Fake -> XXXX

After allocating `three`:
> HEAD -> Fake -> XXXX

After allocating `STACK`:
> HEAD -> XXXX

Now `STACK` will point to `Fake.fd` (not `Fake`) and we can use this pointer to overwrite the stack frame for more hacking.

## Important note
- Please remember that the `STACK` pointer will point to `Fake.fd`, not `Fake`.
- `one` pointer is set to point to `Fake` (the begining of `Fake` chunk), not point to `Fake.fd`. **The machanism of fd pointer is different in various type of bins**.
    > For example, fd pointer of chunk in `tcache_bin` is point to fd pointer of next chunk instead of chunk's header.
- The `Fake.size` must be equal to size of chunk `a` (formally, equal to size of any chunk in same fastbin). Read `calc_tcache_idx.md` to know how to calculate this size.
- The `PREV_INUSE` flag of `Fake.size` (in this situation) is not important, so `Fake.size` can be set to either 112 or 113.
- The `XXXX` in fastbin after *hijacking* can be control by set `Fake.fd` to the suitable address. You can make your exploitation better by set `Fake.fd = c` (`c` is another chunk which same size) and set `c.fd = TAIL`, then the stage of fastbin will be:
    > HEAD -> a -> Fake -> c -> TAIL (PERFECT !!!)
