 # Calculate index of tcache chunk
> LongChampion, 12/03/2020

According to [CTF Wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/glibc-heap/implementation/tcache/):
> Tcache is a technique introduced after `glibc 2.26`, the purpose is to improve the performance of heap management. But while improving performance, **it has abandoned a lot of security checks**, so there are many new ways to use it.

After a few minutes of Google-fu, I conclude that:
1. There are 64 singly-linked tcache_bins per thread by default (index from 0 to 63).
2. Chunksizes are from 32 to 1040 (16 to 520 on x86) bytes, in 16 (8) byte increments.
3. A single tcache_bin contains at most 7 chunks by default.

The first and the last property are "hard-coded", but I think I can explain the second property. The formular in `calc_tcache_idx.c` is a bit terrible, so I simplify it as below:
- Firstly, determine `SIZE_SZ` and `MALLOC_ALIGNMENT`: usually, `SIZE_SZ` is equal to `sizeof(long)` in C++ (8 bytes in x64, 4 bytes in x86) and `MALLOC_ALIGNMENT = 2 * SIZE_SZ`
- Secondly, let `x` is the malloc request size (`x` is argument you pass to `malloc`)
- Then, the adjusted size `t` is the smallest integer, which is not less than `x + SIZE_SZ` and is multiples of `MALLOC_ALIGNMENT`.
- Finally, if `t` is smaller than `MIN_CHUNK_SIZE = 4 * SIZE_SZ`, then `t` is set to `MIN_CHUNK_SIZE`.

For example, in x64 system, `malloc(30)` will be process as: `30 + 8 = 38 -> adjust to 48 -> tcache_bin[1]`.

## Summary:
| index |    chunk size x64   |   chunk size x86  |
|:-----:|:-------------------:|:-----------------:|
|   0   |   32 + 0 * 16 = 32  |  16 + 0 * 8 = 16  |
|   1   |   32 + 1 * 16 = 48  |  16 + 1 * 8 = 24  |
|  ...  |         ...         |        ...        |
|   63  | 32 + 63 * 16 = 1040 | 16 + 63 * 8 = 520 |
