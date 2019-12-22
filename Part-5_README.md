# Hitchhiker's Guide to Debugging C Programs

This is going to be a massive guide to helping you debug your C programs. There are different levels that you can check errors and we will be going through most of them. Feel free to add anything that you found helpful in debugging C programs including but not limited to, debugger usage, recognizing common error types, gotchas, and effective googling tips.


# In-Code Debugging

## Clean code

Make your code modular using helper functions. If there is a repeated task (getting the pointers to contiguous blocks in the malloc MP, for example), make them helper functions. And make sure each function does one thing very well, so that you don't have to debug twice.

Let's say that we are doing selection sort by finding the minimum element each iteration like so,

```C
void selection_sort(int *a, long len){
     for(long i = len-1; i > 0; --i){
         long max_index = i;
         for(long j = len-1; j >= 0; --j){
             if(a[max_index] < a[j]){
                  max_index = j;
             }
         }
         int temp = a[i];
         a[i] = a[max_index];
         a[max_index] = temp;
     }

}
```

Many can see the bug in the code, but it can help to refactor the above method into

```C
long max_index(int *a, long start, long end);
void swap(int *a, long idx1, long idx2);
void selection_sort(int *a, long len);
```

And the error is specifically in one function.

In the end, we are not a class about refactoring/debugging your code. In fact, most systems code is so atrocious that you don't want to read it. But for the sake of debugging, it may benefit you in the long run to adopt some practices.

## Asserts!

Use assertions to make sure your code works up to a certain point -- and importantly, to make sure you don't break it later. For example, if your data structure is a doubly linked list, you can do something like `assert(node->size == node->next->prev->size)` to assert that the next node has a pointer to the current node. You can also check the pointer is pointing to an expected range of memory address, not null, ->size is reasonable etc.
The `NDEBUG` macro will disable all assertions, so don't forget to set that once you finish debugging.

Here's a quick example with assert. Let's say that I'm writing code using memcpy

```C
assert(!(src < dest+n && dest < src+n)); //Checks overlap
memcpy(dest, src, n);
```

This check can be turned off at compile time, but will save you **tons** of trouble debugging!

## printfs

When all else fails, print like crazy! Each of your functions should have an idea of what it is going to do (ie find_min better find the minimum element). You want to test that each of your functions is doing what it set out to do and see exactly where your code breaks. In the case with race conditions, tsan may be able to help, but having each thread print out data at certain times could help you identify the race condition.

# Valgrind


Valgrind is a suite of tools designed to provide debugging and profiling tools to make your programs more correct and detect some runtime issues. The most used of these tools is Memcheck, which can detect many memory-related errors that are common in C and C++ programs and that can lead to crashes and unpredictable behaviour (for example, unfreed memory buffers).

To run Valgrind on your program: 

```
valgrind --leak-check=yes myprogram arg1 arg2
```
or 

```
valgrind ./myprogram
```

Arguments are optional and the default tool that will run is Memcheck. The output will be presented in form of 
number of allocations, number of freed allocations, and the number of errors.

**Example**

![Valgrind Example](https://i.imgur.com/ZdBWDvh.png)

Here's an example to help you interpret the above results. Suppose we have a simple program like this: 
```C
  #include <stdlib.h>

  void dummy_function()
  {
     int* x = malloc(10 * sizeof(int));
     x[10] = 0;        // error 1:as you can see here we write to an out of bound memory address
  }                    // error 2: memory leak the allocated x not freed

  int main(void)
  {
     dummy_function();
     return 0;
  }
```

Let's see what Valgrind will output (this program compiles and run with no errors).
```
==29515== Memcheck, a memory error detector
==29515== Copyright (C) 2002-2015, and GNU GPL'd, by Julian Seward et al.
==29515== Using Valgrind-3.11.0 and LibVEX; rerun with -h for copyright info
==29515== Command: ./a
==29515== 
==29515== Invalid write of size 4
==29515==    at 0x400544: dummy_function (in /home/rafi/projects/exocpp/a)
==29515==    by 0x40055A: main (in /home/rafi/projects/exocpp/a)
==29515==  Address 0x5203068 is 0 bytes after a block of size 40 alloc'd
==29515==    at 0x4C2DB8F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==29515==    by 0x400537: dummy_function (in /home/rafi/projects/exocpp/a)
==29515==    by 0x40055A: main (in /home/rafi/projects/exocpp/a)
==29515== 
==29515== 
==29515== HEAP SUMMARY:
==29515==     in use at exit: 40 bytes in 1 blocks
==29515==   total heap usage: 1 allocs, 0 frees, 40 bytes allocated
==29515== 
==29515== LEAK SUMMARY:
==29515==    definitely lost: 40 bytes in 1 blocks
==29515==    indirectly lost: 0 bytes in 0 blocks
==29515==      possibly lost: 0 bytes in 0 blocks
==29515==    still reachable: 0 bytes in 0 blocks
==29515==         suppressed: 0 bytes in 0 blocks
==29515== Rerun with --leak-check=full to see details of leaked memory
==29515== 
==29515== For counts of detected and suppressed errors, rerun with: -v
==29515== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
**Invalid write**: It detected our heap block overrun (writing outside of allocated block)

**Definitely lost**: Memory leakâ€”you probably forgot to free a memory block

Valgrind is a very effective tool to check for errors at runtime. C is very special when it comes to such behavior, so after compiling your program you can use Valgrind to fix errors that your compiler may not catch and that usually happen when your program is running.


# Tsan

ThreadSanitizer is a tool from Google, built into clang (and gcc), to help you detect race conditions in your code. For more information about the tool, see the Github wiki.

Note that running with tsan will slow your code down a bit.

```C
#include <pthread.h>
#include <stdio.h>

int Global;

void *Thread1(void *x) {
    Global++;
    return NULL;
}

int main() {
    pthread_t t[2];
    pthread_create(&t[0], NULL, Thread1, NULL);
    Global = 100;
    pthread_join(t[0], NULL);
}
// compile with gcc -fsanitize=thread -pie -fPIC -ltsan -g simple_race.c
```

We can see that there is a race condition on the variable `Global`. Both the main thread and the thread created with pthread_create will try to change the value at the same time. But, does ThreadSantizer catch it?

```
$ ./a.out
==================
WARNING: ThreadSanitizer: data race (pid=28888)
  Read of size 4 at 0x7f73ed91c078 by thread T1:
    #0 Thread1 /home/zmick2/simple_race.c:7 (exe+0x000000000a50)
    #1  :0 (libtsan.so.0+0x00000001b459)

  Previous write of size 4 at 0x7f73ed91c078 by main thread:
    #0 main /home/zmick2/simple_race.c:14 (exe+0x000000000ac8)

  Thread T1 (tid=28889, running) created by main thread at:
    #0  :0 (libtsan.so.0+0x00000001f6ab)
    #1 main /home/zmick2/simple_race.c:13 (exe+0x000000000ab8)

SUMMARY: ThreadSanitizer: data race /home/zmick2/simple_race.c:7 Thread1
==================
ThreadSanitizer: reported 1 warnings
```

If we compiled with the debug flag, then it would give us the variable name as well.

# GDB


#### Setting breakpoints programmatically

A very useful trick when debugging complex C programs with GDB is setting breakpoints in the source code.

```c
int main() {
    int val = 1;
    val = 42;
    asm("int $3"); // set a breakpoint here
    val = 7;
}
```

```sh
$ gcc main.c -g -o main && ./main
(gdb) r
[...]
Program received signal SIGTRAP, Trace/breakpoint trap.
main () at main.c:6
6	    val = 7;
(gdb) p val
$1 = 42
```



#### Checking memory content


For example,

```c
int main() {
    char bad_string[3] = {'C', 'a', 't'};
    printf("%s", bad_string);
}
```

```sh
$ gcc main.c -g -o main && ./main
$ Cat ZVQï¿½ $
```

```sh
(gdb) l
1	#include <stdio.h>
2	int main() {
3	    char bad_string[3] = {'C', 'a', 't'};
4	    printf("%s", bad_string);
5	}
(gdb) b 4
Breakpoint 1 at 0x100000f57: file main.c, line 4.
(gdb) r
[...]
Breakpoint 1, main () at main.c:4
4	    printf("%s", bad_string);
(gdb) x/16xb bad_string
0x7fff5fbff9cd:	0x63	0x61	0x74	0xe0	0xf9	0xbf	0x5f	0xff
0x7fff5fbff9d5:	0x7f	0x00	0x00	0xfd	0xb5	0x23	0x89	0xff

(gdb)
```

Here, by using the `x` command with parameters `16xb`, we can see that starting at memory address `0x7fff5fbff9c` (value of `bad_string`), printf would actually see the following sequence of bytes as a string because we provided a malformed string without a null terminator.

```0x63 0x61 0x74 0xe0 0xf9 0xbf 0x5f 0xff 0x7f 0x00```

<div align="center">
<a href="https://github.com/akhilesh1806/C-Programming/blob/master/Part-4_README.md">
Back: C Programming, Part 4: Strings and Structs
</a>
| <a href="">
Back: C Programming, Review Questions
</a>
</div>
