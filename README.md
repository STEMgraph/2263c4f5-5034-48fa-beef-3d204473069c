<!---
{
  "id": "2263c4f5-5034-48fa-beef-3d204473069c",
  "teaches": "Parsing Options with `getopt`",
  "depends_on": ["5adc24d3-4054-453e-80dd-30a820be8ed3"],
  "author": "Stephan Bökelmann",
  "first_used": "2025-06-26",
  "keywords": ["C", "getopt", "optarg", "main", "command-line"]
}
--->

# Parsing Options with `getopt`

> In this exercise you will learn how to use the `getopt` (or `getopt_long`) library in C for parsing command-line switches and parameters. Furthermore, we will explore how to implement required options, optional parameters, and always include a `--help` output for user guidance.

## Introduction

When you run a C program from the command line, the operating system collects everything you type, splits it into separate tokens based on spaces and quoting rules, and places those tokens into an array called `argv`, with `argc` indicating the total count. Manually iterating over `argv` works for simple cases, but as soon as you need to handle flags (like `-q`), options with values (like `-o output.txt`), or both together, your code can become error‑prone and hard to maintain.

The POSIX `getopt` function (and its GNU cousin `getopt_long`) streamlines this process by interpreting a format string that declares which options your program accepts. It hides the details of moving through `argv`, checking for hyphens, and extracting option arguments. Instead, you call `getopt` in a loop, and it returns one option at a time until no more remain.

Key concepts:

* **`optind`**: An index into `argv` that `getopt` updates to point to the next argument to process. After option parsing, `optind` tells you where positional parameters begin.
* **`optarg`**: A pointer to the current option’s argument string, but only when the option requires an argument.
* **`struct option`**: A way to define long options (e.g., `--help`) in an array, similar to how you might define an array of strings. Each element describes one option’s name, whether it takes an argument, and what short character it corresponds to.

Even if you haven’t encountered `struct`s yet, think of each element in the `struct option` array like a row in a table: you give a column for the long name, a column for argument requirement, and so on. You can use this pattern without needing to define your own structs just yet.

In this exercise, you will step through three programs:

1. **`switches.c`** — Only boolean switches (`-q` / `--quiet`).
2. **`params.c`** — Only positional parameters (no switches).
3. **`filetool.c`** — A full example combining switches, parameters, and a mandatory `--help` option.

By building these in order, you’ll see how to incrementally add complexity and leverage `getopt` to keep your code clean.

### Further Readings and Other Sources

* Michael Kerrisk, *The Linux Programming Interface*, 2010.
* GNU C Library Manual: Command-Line Argument Syntax – [https://www.gnu.org/software/libc/manual/html\_node/Argument-Syntax.html](https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)
* YouTube: “Parsing Command-Line Options in C with getopt” – [https://www.youtube.com/watch?v=XB4MIexjvY0](https://www.youtube.com/watch?v=XB4MIexjvY0)

## Tasks

### Task 1: Switches Only

In this task, you will create `switches.c`, a minimal program that detects a single flag `-q` (quiet mode). This will introduce you to calling `getopt`, interpreting its return values, and using a simple `while` loop to consume all options. You will see how `getopt` automatically stops at the first non-option argument, updating `optind` so you could continue processing further parameters if needed.

1. Create `switches.c` with the following code:

   ```c
   #include <stdio.h>
   #include <unistd.h>  // for getopt

   int main(int argc, char *argv[]) {
       int opt;
       int quiet = 0;

       // Loop through options until getopt returns -1
       while ((opt = getopt(argc, argv, "q")) != -1) {
           switch (opt) {
               case 'q':
                   quiet = 1;
                   break;
               default:
                   // getopt prints an error message for unrecognized options
                   break;
           }
       }

       // At this point, optind points to the first non-option argument
       printf("quiet=%d (optind=%d)\n", quiet, optind);
       return 0;
   }
   ```

2. Compile and experiment:

   ```bash
   gcc -o switches switches.c
   ./switches -q
   ./switches
   ./switches -x    # Observe how getopt handles an unknown flag
   ```

   Notice how `optind` changes based on which arguments are recognized.

### Task 2: Positional Parameters Only

Next, you will write `params.c`, which ignores any switches and prints every argument that follows the program name. While this doesn’t use `getopt`, it reinforces the idea that arguments at indices `1` through `argc-1` are simply strings provided on the command line.

1. Create `params.c`:

   ```c
   #include <stdio.h>

   int main(int argc, char *argv[]) {
       // Loop from argv[1] to argv[argc-1]
       for (int i = 1; i < argc; i++) {
           printf("Input file: %s\n", argv[i]);
       }
       return 0;
   }
   ```

2. Compile and test:

   ```bash
   gcc -o params params.c
   ./params file1.txt file2.dat extra
   ```

   Observe that all tokens become "Input file" lines.

### Task 3: Combining Options, Parameters, and `--help`

Finally, you will build `filetool.c`, combining quiet mode, an output filename option, and support for `--help`. This program demonstrates a real-world pattern: define your options in a `struct option` array, loop with `getopt_long`, then handle remaining positional arguments. A clear `--help` message improves usability and serves as built-in documentation.

1. Create `filetool.c` and start with headers and a verbose comment on `long_options`:

   ```c
   #include <stdio.h>
   #include <stdlib.h>
   #include <getopt.h>

   /* long_options array synopsis:
      Each element has: .name ("quiet"), .has_arg (no_argument or required_argument),
      .flag (NULL if we want getopt_long to return val), and .val (the shortcut char).
      Think of this as an array of 4-column rows defining each option.
      You can use it without full struct knowledge.
   */
   static struct option long_options[] = {
       {"quiet",   no_argument,       0, 'q'},  // --quiet or -q
       {"output",  required_argument, 0, 'o'},  // --output=<file> or -o <file>
       {"help",    no_argument,       0, 'h'},  // --help or -h
       {0, 0, 0, 0}                           // termination entry
   };
   ```

2. In `main`, declare parsing variables and loop:

   ```c
   int quiet = 0;
   char *outfile = NULL;
   int opt, option_index;

   // Loop over options
   while ((opt = getopt_long(argc, argv, "qo:h", long_options, &option_index)) != -1) {
       switch (opt) {
           case 'q':
               quiet = 1;
               break;
           case 'o':
               outfile = optarg; // optarg points to the string following -o
               break;
           case 'h':
               printf("Usage: %s [-q|--quiet] [-o|--output <file>] [--help] <inputs>...\n", argv[0]);
               exit(EXIT_SUCCESS);
           default:
               fprintf(stderr, "Unknown option '%c'. Use --help.\n", optopt);
               exit(EXIT_FAILURE);
       }
   }
   ```

3. After option parsing, process positional arguments:

   ```c
   // Report settings
   if (quiet) printf("Quiet mode ON\n");
   if (outfile) printf("Output file: %s\n", outfile);

   // Remaining argv entries are inputs
   for (int i = optind; i < argc; i++) {
       printf("Input: %s\n", argv[i]);
   }
   ```

4. Compile and run various scenarios:

   ```bash
   gcc -o filetool filetool.c
   ./filetool --help
   ./filetool -q -o out.txt in1.txt in2.txt
   ./filetool in1.txt --quiet in2.txt
   ```

## Advice

By following these steps, you’ll gain a clear understanding of how `getopt` and `getopt_long` work internally while keeping your code organized. Remember that the `--help` option is essential: it documents your program’s interface for users. Continue experimenting by adding new flags, making arguments optional, or exploring libraries like `argp` for more advanced parsing features. Always compile often in `vim` and test each change to reinforce learning.
