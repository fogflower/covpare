# covpare

Simple tool for comparing coverage data generated by `gcov` or `llvm-cov` between runs.

## Why does this exist?

Because I couldn't find any other open-source tools for diffing `gcov`-generated coverage data.  There's lots of things to turn it into pretty HTML and see what coverage you get for *one* run, but nothing to compare runs.

Additionally, the build-time instrumentation is pretty fast (compared to DBI tracing).

## Requirements

- LLVM (tested) or GCC (untested) build chain
- MongoDB (used for storage)
- Python 2.7

## Usage

There's an example in `test` that shows the usage, but the general steps are shown here.

### Build Software

Build your software with the `--coverage` flag.  It's accepted by any recent `gcc`, as well as `llvm`.  Quick-and-easy ways to bolt the flag onto an existing build process are:

    # ./configure && make -style builds
    export CFLAGS='-O0 --coverage'
    export CXXFLAGS='-O0 --coverage'
    export LDFLAGS='--coverage'

    # debian builds via dpkg-buildpackage or debian/rules
    export DEB_CFLAGS_SET='--coverage'
    export DEB_LDFLAGS_SET='--coverage'

### Run the software

Run it however it would normally be run.  Ideally, you'll have two ideas that you want to compare.

When running the software, the coverage instrumentation added at build-time will generate `gcda` files.  These are not compatible between versions of `gcc` (and thus versions of `gcov`), nor between `gcc` and `llvm`.

### Parse .GCDA

`gcov` or `llvm-gcov` will parse the `.gcda` files into various textual representations of the coverage data.  This tool, `covpare`, likes all the bells and whistles to be on, so the command you'll use will look like:

    llvm-cov-3.5 -a -b -c -stats -f hello.gcda

### Parse .GCOV

Now we've got a bunch of .gcov files which have been emmitted.  These are parsed by the `parse.py` script and stored in MongoDB.  Choose an identifier for this set of coverage.  It must be a valid MongoDB collection name (alphanumeric and underscores).

    python parse.py <IDENTIFIER> hello.cc.gcov

The input files should look like this:

    function main called 1 returned 100% blocks executed 87%
            -:   11:int main(int argc, char** argv)
            -:   12:{
            1:   13:    int x = 3;
            -:   14:
            1:   15:    if(argc == 1) {
            1:   15-block  0
    branch  0 taken 0
    branch  1 taken 1
        #####:   16:        printf("Hello, world!\n");
        #####:   17:    }
        $$$$$:   17-block  0
            -:   18:
    ...
    function _GLOBAL__I_a called 1 returned 100% blocks executed 100%
            1:   25:}
            1:   25-block  0


### Rinse and Repeat

In order to `diff` anything you've got to have multiple runs, right?

Delete all of the `.gcda` and `.gcov` files, and run the software again to test whatever differences you want to see.

In the case of the example `hello` binary provided, it will print "hello world" with no arguments, or loop through all of the arguments provided, saying "hello" to each.

### Compare!

This is what you're here for, right?

    compare.py --arguments <IDENTIFIER_A> <IDENTIFIER_B>

What you want to see, `--arguments`, is documented via `compare.py --help`.
These examples were taken from the provided test binary.

#### Function Diff

In the test binary, `greet` has 100% more coverage in the second, right-hand run.  Main has 12% better coverage.

```
// $ python compare.py left right --function-diff
+100% right: greet(char*)
+12% right: main [1 75%] [1:87%]
```

#### Call Diff

In the test binary, `greet` is called zero times on the left-hand-side, and twice on the right-hand side.

```
// $ python compare.py left right --call-diff
0 2 greet(char*)
```

#### Line Diff

Here we can see the specific lines that are executed, and their coverage data.  The second column contains the number of times each line was hit.  For example, the `for` line was only executed once in the first run, but three times in the third run.  Likewise, the `Hello world` printf is never executed in the second run.

```c
// $ python compare.py left right --line-diff
hello.cc:4    |        1 1 | void* x = malloc(30); 
hello.cc:5    |        0 0 |  
hello.cc:6    |        0 0 | void greet(char* name) 
hello.cc:7    |        0 0 | { 
hello.cc:8    |        0 1 |     printf("Hello, %s\n", name); 
hello.cc:9    |        0 1 | } 
hello.cc:10   |        0 0 |  
hello.cc:11   |        0 0 | int main(int argc, char** argv) 
hello.cc:12   |        0 0 | { 
hello.cc:13   |        1 1 |     int x = 3; 
hello.cc:14   |        0 0 |  
hello.cc:15   |        1 1 |     if(argc == 1) { 
hello.cc:16   |        1 0 |         printf("Hello, world!\n"); 
hello.cc:17   |        1 0 |     } 
hello.cc:18   |        0 0 |  
hello.cc:19   |        1 3 |     for(int i = 1; i < argc; i++) 
hello.cc:20   |        0 0 |     { 
hello.cc:21   |        0 2 |         greet(argv[i]); 
hello.cc:22   |        0 2 |     } 
hello.cc:23   |        0 0 |  
hello.cc:24   |        1 1 |     return 0; 
```