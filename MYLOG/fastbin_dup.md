# Fastbin duplicate
> LongChampion, 12/03/2020

Fastbin are LIFO (as a stack), so we can use *double free* to make malloc return an allocated chunk.  
Our tictac is (`SIZE` must fall into fastbin size):
```
a = malloc(SIZE)
b = malloc(SIZE)

free(a)
free(b)
free(a)

d = malloc(SIZE)
e = malloc(SIZE)
f = malloc(SIZE)
```
Note, that we can't always free `a` two time consecutively, because new version of **LIBC** will compare the chunk to be free with the last chunk in the bin, if there are identical, **LIBC** knows you are hacking and terminates the program immediately. To bypass this check, we simple free another chunk (chunk `b`) before free chunk `a` again.  
The stage of fastbin during the process can be represent as (insert and remove happen at the HEAD of list):

0. Fastbin is empty
    > HEAD -> TAIL
1. `a` is freed
    > HEAD -> a -> TAIL
2. `b` is freed to bypass double free checking
    > HEAD -> b -> a -> TAIL
3. `a` is freed again
    > HEAD -> a -> b -> a -> TAIL
4. `malloc` is called: `d = a`
    > HEAD -> b -> a -> TAIL
5. `malloc` is called: `e = b`
    > HEAD -> a -> TAIL
6. `malloc` is called: `f = a` (Hacked!)
    > HEAD -> TAIL

It's simple that if we control content of chunk `d`, we also control content of chunk `f` and vice versa.  
But further, we can lead to *fastbin attack* if we modify chunk `d` by the right way.

# Reference
[Heap Exploitation](https://heap-exploitation.dhavalkapil.com/attacks/double_free.html)