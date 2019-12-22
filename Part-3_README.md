## What common mistakes do C programmers make?

# Memory Mistakes

## String Constants are Read-Only
```C
char array[] = "Hi!"; // array contains a mutable copy 
strcpy(array, "OK");

char *ptr = "Can't change me"; // ptr points to some immutable memory
strcpy(ptr, "Will not work");
```
String literals are character arrays stored in the code segment of the program, which is immutable. Two string literals may share the same space in memory. An example follows:

```C
char *str1 = "Brandon Chong is the best TA";
char *str2 = "Brandon Chong is the best TA";
str1 == str2; // true
```
The strings pointed to by `str1` and `str2` may actually reside in the same location in memory.

Char arrays, however, contain deep copies of string literals assigned to them, and different arrays represent different memory locations. These following char arrays do not reside in the same place in memory.

```C
char arr1[] = "Brandon Chong didn't write this";
char arr2[] = "Brandon Chong didn't write this";
arr1 == arr2;         // false
&arr1[0] == &arr2[0]; // false
```

## Buffer Overflow / Underflow
```C
int i = 10, array[10];
for (; i >= 0; i--) array[i] = i;
```
C does not perform bounds-checking on array accesses. The above example writes into `array[10]` which is outside the array bounds. This can corrupt other variables on the stack as well as the very implementation of the call stack, exposing your program to attacks from hackers. In practice, these overflows often result from using unsafe library calls or putting the wrong size limit on a safer library call.

```C
gets(array); // Let's hope the input is shorter than my array! (NEVER use gets)
fgets(array, 4096, stdin); // Whoops 
```

## Handling Pointers to Out-of-Scope Automatic Variables
```C
int *f() {
    int result = 42;
    static int imok;
    int *p;
    {
      int x = result;
      p = &x;
    }
    //imok = *p;      // Not OK: x has already gone out of scope
    //return &result; // Not OK: result will go out of scope after the function returns 
    return &imok;     // OK - static variables are not on the stack
}
```
Automatic variables are bound to stack memory only as long as they are in scope. After they go out of scope, the data stored at their memory addresses becomes undefined. Static variables reside in the data segment, which is safe to access even when those variables are not in scope.

## `sizeof(type *)` versus `sizeof(type)` 
```C
struct User {
   char name[100];
};
typedef struct User user_t;

user_t *user = (user_t *) malloc(sizeof (user_t *));
```
In the above example, we needed to allocate enough bytes for the struct. Instead, we allocated enough bytes to hold a pointer. Writing to the user pointer might corrupt the heap. The correct code is shown below.
```C
struct User {
   char name[100];
};
typedef struct User user_t;

user_t *user = (user_t *) malloc(sizeof (user_t));
```

## Strings Require `strlen(s)+1` Bytes

Every string must have a null byte after the last actual character. To store the string <code>"Hi"</code> it takes 3 bytes: <code>[H] [i] [\0]</code>.

```C
char *strdup(const char *input) {     /* return a copy of 'input' */
    char *copy;
    copy = malloc(sizeof (char *));   /* nope! this allocates space for a pointer, not a string */
    copy = malloc(strlen(input));     /* Almost...but what about the null terminator? */
    copy = malloc(strlen(input) + 1); /* That's right. */
    strcpy(copy, input);              /* strcpy will provide the null terminator */
    return copy;
}
```

## Failing to Initialize Memory
```C
void myfunct() {
    char array[10];
    char *p = malloc(10);
    printf("%s %s\n", array, p);
}
```
Automatic (stack) variables and heap memory obtained with `malloc` are not initialized to zero by default. The function above results in undefined behavior.

## Double-free
```C
char *p = malloc(10);
free(p);
//  .. later ...
free(p); 
```
It is an error to free the same heap memory twice.

## Dangling Pointers
```C
char *p = malloc(10);
strcpy(p, "Hello");
free(p);
//  .. later ...
strcpy(p,"World"); 
```
Accessing freed memory results in undefined behavior. A defensive programming practice is to set pointers to NULL as soon as the memory is freed, since there is no other way to reliably check if a pointer points to a valid address. The following macro accomplishes this.

```C
#define safer_free(p) {free(p); (p) = NULL;}
```

## Forgetting to Copy `getline` Buffer
```C
#include <stdio.h>
  
int main(void){
  char *line = NULL;
  size_t linecap = 0;
  char *strings[3];

  // assume stdin contains "1\n2\n\3\n"
  for (size_t i = 0; i < 3; ++i)
    strings[i] = getline(&line, &linecap, stdin) >= 0 ? line : "";

  // this prints out "3\n3\n\3" instead of "3\n\2\n1\n"
  for (size_t i = 3; i--;) // i=2,1,0
    printf("%s", strings[i]);
}
```
Since `getline` reuses a buffer, all pointers in the `strings` array are actually pointing to the same memory. We can fix this by setting the assignment of `strings[i]` to a deep copy of the buffer.
```C
   strings[i] = getline(&line, &linecap, stdin) >= 0 ? strdup(line) : "";
```
Fun fact: providing "1\n123456789abcdef\n3\n" to the broken version of the program might cause it to print out "3\n3\n1\n" instead of "3\n3\n3\n". Now why might that be? 
*Hint: use Valgrind*

# Logic and Program Flow Mistakes
## Forgetting `break` after `case` 
```C
int flag = 1; // Will print all three lines.
switch (flag) {
case 1: printf("I'm printed\n");
case 2: printf("Me too\n");
case 3: printf("Me three\n");
}
```
Case statements without a break will just continue onto the code of the next case statement. The correct code is shown below. The break for the last statement is unnecessary because there are no more cases to be executed after the last one.
```C
int flag = 1; // Will print only "I'm printed\n"
switch (flag) {
case 1: 
    printf("I'm printed\n");
    break;
case 2: 
    printf("Me too\n");
    break;
case 3: 
    printf("Me three\n");
    break; //unnecessary
}
```

## Assignment vs Equality Check

```C
int answer = 3; // Will print out the answer.
if (answer = 42) { printf("I've solved the answer! It's %d", answer); }
```
Compilers will usually warn you about this mistake. If you really want to perform an assignment, add an extra pair of parentheses to suppress these warnings.
```C
ssize_t x;
if ( (x = read(somefd, somebuf, somenum)) ){
  // do something
}
```

## Undeclared or Incorrectly Prototyped Functions

```C
#include <stdio.h>
int main(void){
  int start = time();
  printf("%d\n", start);
}
```
The library call `time` actually takes a parameter (a pointer to some memory that can receive the `time_t` structure). The compiler might not catch this error because the programmer did not provide a valid function prototype by including time.h. For this reason, calling undeclared functions is illegal in C99 and beyond.

## Extra Semicolons

Semicolons after `for` and `while` statements will cause them to be interpreted as "empty loops".

```C
int i;
for (i = 0; i < 5; i++);{
  printf("I'm printed once\n");
}
while (i < 10); // program goes into infinite loop
  i++;          // this code is never executed
```

Adding semicolons adjacent to each other in code blocks is perfectly legal and is ignored by the compiler. You can also omit components of a `for` statement by putting nothing between semicolons or between semicolons and parentheses.
```C
int i = 0;
for (; i++ < 5;) { // i=1,2,3,4,5
    printf("%d\n", i);;;;;;;;;;;;;
}
```

# Other Gotchas
## C Preprocessor macros and precedence

Preprocessing is an operation performed **before** the program is actually compiled. It is nothing but text substitution, i.e. copy-and-paste. Consider the following code.
```C
#define BADD(x,y) x+y
char buffer[BADD(5,5)*2];
```

After preprocessing, the code will look exactly like this.
```C
char buffer[5+5*2];
```

Notice that the buffer takes up 15 bytes instead of 20, since multiplication has higher precedence than addition. To prevent this kind of behavior, make sure to surround all macro variables as well as the entire macro expression with parentheses.
```C
#define ADD(x,y) ((x)+(y))
```

## C Preprocessor Macros and Side-Effects
```C
#define min(a,b) ((a)<(b) ? (a) : (b))
int x = 4;
if (min(x++, 100)) printf("%d is six", x);
```
The conditional expression effectively expands to `x++ < 100 ? x++ : 100`, which results in `x` being incremented twice. There is no good way to prevent these kinds of side effects when using Standard C macros. But GNU C provides some workarounds (only works when using GCC).

## `sizeof(type[])` vs `sizeof(type *)`
```C
#define ARRAY_LENGTH(A) (sizeof((A)) / sizeof((A)[0]))
int fixed_length_array[10]; // ARRAY_LENGTH(fixed_length_array) = 10
int *dynamic_array = malloc(10); // ARRAY_LENGTH(dynamic_array) = 2 or 1
```
If we have a declared array variable like `fixed_length_array`, the `sizeof` operator yields the bytes that array takes up, and dividing this size by the size of the first element yields the number of elements in the array. Unfortunately, the size of a pointer is always the same (8 or 4 bytes), no matter the size or type of the array to which it points. Only compile-time declared array variables will expose the true array size through `sizeof`.

## `sizeof` and Side-Effects

```C
int a = 0;
size_t size = sizeof(a++);
printf("size: %lu, a: %d", size, a);
```
This code prints out the following.
```C
size: 4, a: 0
```
The expression passed into `sizeof` is not actually evaluated at runtime in most cases, since the type (hence the size) of the expression can be calculated at compile time. Though there are exceptions in the case of [C99 variable-length arrays](http://port70.net/~nsz/c/c11/n1570.html#6.5.3.4p2), since their sizes are determined at runtime.

<div align="center">
<a href="https://github.com/akhilesh1806/C-Programming/blob/master/Part-2_README.md">
Back: C Programming, Part 2: Text Input And Output
</a> |
<a href="https://github.com/akhilesh1806/C-Programming/blob/master/Part-4_README.md">
Next: C Programming, Part 4: Strings and Structs
</a>
</div>
