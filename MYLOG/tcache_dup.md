# Tcache duplicate
> LongChampion, 21/03/2020

This attack work as `fastbin_dup`:
```
#include <stdlib.h>

int main()
{
    void *a = malloc(0x100);
    free(a);
    free(a);
    void *a1 = malloc(0x100);
    void *a2 = malloc(0x100);
}
```
Unlike fastbin, early version of tcache is lack of many security checks, include *double free checking*. So we even no need to free a temporary chunk to bypass this check (we can't bypass the check that doesn't exits).

## Note
Because the attack is very simple, it have been patched and no longer work against newer version of **GLIBC**.
You should read `calc_tcache_idx.md` for more information about tcache_bin.
