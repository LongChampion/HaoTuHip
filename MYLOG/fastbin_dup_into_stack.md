# Fastbin duplicate into STACK
> LongChampion, 12/03/2020

> Please read `fastbin_dup.md` to know how to duplicate the chunk in fastbin linked list.

My code:
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    struct Chunk FAKE;
    void *a = malloc(100);
    void *b = mallloc(100);

    free(a), free(b), free(a);

    void *one = malloc(100);
    void *two = malloc(100);

    one = &FAKE;
    FAKE.size = 112;

    void *three = malloc(100);
    void *STACK = malloc(100);
}
```
The ideal is simple: we use *double free* to duplicate chunk `a`, then call `malloc` to receive it again. When we modify content of `one`, we also modify content of `three` (although which haven't allocated yet). The stage of fastbin before *hijacking* is:
> HEAD -> a -> TAIL

After *hijacking*:
> HEAD -> a -> FAKE -> XXXX

After allocating `three`:
> HEAD -> FAKE -> XXXX

After allocating `STACK`:
> HEAD -> XXXX

Now `STACK` will point to FAKE.fd (not FAKE) and we can use this pointer to overwrite the stack frame for more hacking.

# IMPORTANT NOTE
- Please remember that the `STACK` pointer will point to `FAKE.fd`, not `FAKE`.
- `one` pointer is set to point to `FAKE` (the begining of `FAKE` chunk), not point to `FAKE.fd`. **The machanism of fd pointer is different in various type of bins**.
    > For example, fd pointer of chunk in `tcache_bin` is point to fd pointer of next chunk instead of chunk's header.
- `FAKE.size` must be equal to size of chunk `a` (formally, equal to size of any chunk in same fastbin). Read `calc_tcache_idx.md` to know how to calculate this size.
- `PREV_INUSE` flag of `FAKE.size` (in this situation) is not important, so `FAKE.size` can be set to either 112 or 113.
- `XXXX` in fastbin after *hijacking* can be control by set `FAKE.fd` to the suitable address. You can make your exploitation better by set `FAKE.fd = c` (`c` is another chunk which same size) and set `c.fd = TAIL`, then the stage of fastbin will be:
    > HEAD -> a -> FAKE -> c -> TAIL (PERFECT !!!)
