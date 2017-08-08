# Debugopt

## Introduction

Debugopt is a tool for debugging optimized binary code by switching to the unoptimized version. Debugopt is consisted of the following components:

+ debugopt-clang
The main driver for compiling source code (in C/C++)
+ debugopt-gdb
The debugger driver for debugging program compiled by debugopt-clang

```diff
- The debugopt binary for x86-64 is available in `debugopt.tar.gz`
```

## Usage

### debugopt-clang
debugopt-clang works exactly the same with standard clang. When the program is compiled with both option -O2/-O3 and -g, debugopt-clang automatically generate the unoptimized functions. debugopt-clang also provides two extra option:

| Command      | Explanation |
| ------------ | --- |
| --savetemp   | Saves all the temporal generated files. Normally those temporal generated files are automatically removed after the compiling. |
| --separation | When the unoptimized functions are linked into a separated shared library. |

### debugopt-gdb

debugopt-gdb works exactly the same with standard gdb. debugopt-gdb supports several extra commands:

| Command             | Explanation
| ------------------- | ---
| -b/-tb/-hb/-thb/-rb | The extra commands supported by debugopt-gdb. They have the same syntax and semantic with the intrinsic b/tb/hb/thb/rb commands. But they are used to insert debugopt breakpoints that locate in the unoptimized functions.
| -d                  | The extra commands supported by debugopt-gdb. It has the same syntax and semantic with the intrinsic d commands. But it is for deleting debugopt breakpoints.
| -di/-en	          | Disable/enable debugopt breakpoints. They have the same synax with the intrinsic gdb command disable/enable.
| -s/-n               | The debugopt single stepping/next command for stepping into/stepping over a function with unoptimized code.

## Example

### The example program
We provide an example usage in the test folder. The content of main.c is the following

```
$ cd test
$ cat -n main.c
 1	int printf(const char *fmt, ...);
 2	
 3	void foo(int n, int m) {
 4	    int i;
 5	    for (i = 1; i < m; i++) {
 6	        int k = n * 3;
 7	        int v = k * k;
 8	        n = n + i;
 9	        printf("%s %d\n", __PRETTY_FUNCTION__, v);
10	    }
11	}
12	
13	void (*pfoo)(int, int) = foo;
14	
15	int main() {
16	    pfoo(30, 5);
17	    return 0;
18	}
19	
```
		
### In normal debugging

When the program is compiled with full optimization by gcc and debugged by gdb

```
$ gcc main.c -O3 -g -o main
$ gdb main
(gdb) b main.c : 9
Breakpoint 1 at 0x40056c: file main.c, line 9.
(gdb) r
Starting program:
/.../main

Breakpoint 1, foo (n=30, m=5) at main.c:9
9               printf("%s %d\n", __PRETTY_FUNCTION__, v);
(gdb) p n
$1 = 30
```
		
When breakpoint 1 is hit at the first time, the value of `n` should be 31, instead of 30. The reason is that the optimization makes the debugging information inaccurate.

### In debugopt's debugging

When the program is compiled with full optimization and then debugged by debugopt

```
$ debugopt-clang main.c -O3 -g -o main.debugopt
$ debugopt-gdb main.debugopt
=================================================== Debugopt =============================================
(gdb) -b main.c : 9
(gdb) r
Starting program:
/.../main.debugopt
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
$1 = 0

Breakpoint 1, [foo] (n=31, m=5) at main.c:9
9               printf("%s %d\n", __PRETTY_FUNCTION__, v);
(gdb)p n
$2 = 31
```
		
When debugged by debugopt, the value of `n` is correct, and the current running of function `foo` is the unoptimized code with accurate debugging information.

## Requirement
The current release is based on x86_64 platform. The gdb version should be over 7.7 which supports the python operation interface. The release contains the clang executable. So an installed clang on the host machine is not required.