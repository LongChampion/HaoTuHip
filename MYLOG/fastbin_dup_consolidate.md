# Fastbin duplicate using `malloc_consolidate`
> LongChampion, 13/03/2020

If you have read `fastbin_dup.md` and `fastbin_dup_into_stack.md`, you known how to bypass *double free checking* of fastbin. But now, we have a new method to bypass it.  
My code (very similar to the code in `fastbin_dup_consolidate.c`):
```
#include <stdlib.h>

int main()
{
    void *a = malloc(0x40);
    void *b = malloc(0x40);

    free(a);
    void *c = malloc(0x400);
    free(a);

    void *one = malloc(0x40);
    void *two = malloc(0x40);
}
```
Here, after free `a`, this chunk is put into fastbin, and to free it again, we allocate a large chunk (size of this chunk fall into largebin size). According to `fastbin_dup_consolidate.c`, this `malloc` will call `malloc_consolidate`, and because our request size is not fastbin size, `malloc_consolidate` will move `a` to unsorted_bin. But if you use GDB (you must install some extension such as [GEF](https://github.com/hugsy/gef)) to trace, you see that `a` is moved to smallbin instead of unsorted_bin. WTF happened? Is there is mistake by the author?  
> NO, the author is right, `malloc_consolidate` really move `a` to unsorted_bin, but after that, `malloc` (in particular: the code of `_int_malloc`) will process the unsorted_bin, then move `a` from unsorted_bin to smallbin as you have seen.

Now `a` is no longer exits in fastbin, so we can free it again and the next two `malloc` calls will return `a` as we expect.

# MORE INFO
Our purpose is to duplicate `a`, so can we remove the following line from our code:
> `void *b = malloc(0x40);`

The answer is NO. This line is very important. If we remove it, the result will be very different: `c` and `one` are both point to chunk `a`, and `two` will point to another chunk. Look at this [HEAP_MAP](https://raw.githubusercontent.com/cloudburst/libheap/master/heap.png) and follow `malloc` flow when request size fall into largebin size, you will reach `malloc_consolidate` block. In this block, these is a question **next chunk top?**, and if the answer is *yes*, `malloc_consolidate` will merge the current chunk with top chunk. This is reason why we need chunk `b`. (Try to understand it by yourself)
