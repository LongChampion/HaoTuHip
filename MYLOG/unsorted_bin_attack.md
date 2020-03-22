# Unsorted_bin attack
> LongChampion, 19/03/2020

Initially, I don't think this is an attack, this is only the trick to overwrite one address in memory with a large value. But after a bit research, I see that the value used to overwrite is very interesting: this is a address of `main_arena + 88`. Why this address is interesting? If we overwrite this value to somewhere then read it, we can leak **GLIBC** base address.
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

int main()
{
    long long Value = 0x1234567890abcdef;

    void *victim = malloc(0x100 - 8);
    void *pivot = malloc(0x100 - 8);

    free(victim);
    struct Chunk *header = (struct Chunk *) (victim - 0x10);
    header->bk = (struct Chunk *) (&Value - 2);

    void *STACK = malloc(0x100 - 8);
}
```
Look at my `unsorted_bin_into_stack.md`:
> 8 bytes at address `Fake.bk + 0x10` will be overwrite

This is how this trick work, set `header->bk` to offset 0x10 before variable `Value`, then call `malloc` with exact request size, `Value` will be overwrite to `main_arena+88`. End of trick!

## Note
- At this time of writing, this trick is no longer work with latest version of **GLIBC** (my **GLIBC** version is 2.31)
- Another way to use this trick is to overwrite the `global_max_fast` in **GLIBC** for further fastbin attack.
- `Value` is define as `long long`, so `&Value - 2` will return the address at offset 0x10 before `Value`, don't be confuse with `&Value - 0x10`.
