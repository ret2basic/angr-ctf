# Fuzzing Xpdf

For this exercize we will fuzz **Xpdf PDF viewer**. The goal is to find a crash/PoC for **CVE-2019-13288** in XPDF 3.02. In this exercise, we will learn:

- Compiling a target application with instrumentation
- Running a fuzzer (afl-fuzz)
- Triaging crashes with a debugger (GDB)

## Setup

Create a new directory for our project:

```shell
cd $HOME && mkdir fuzzing_xpdf && cd fuzzing_xpdf/
```

Download and extract Xpdf 3.02:

```shell
wget https://dl.xpdfreader.com/old/xpdf-3.02.tar.gz && tar -xvzf xpdf-3.02.tar.gz
```

Build Xpdf:

```shell
cd xpdf-3.02 && ./configure --prefix="$HOME/fuzzing_xpdf/install/" && make && make install
```

Test Xpdf:

```shell
cd $HOME/fuzzing_xpdf && mkdir pdf_examples && cd pdf_examples && wget https://github.com/mozilla/pdf.js-sample-files/raw/master/helloworld.pdf && wget http://www.africau.edu/images/default/sample.pdf && wget https://www.melbpc.org.au/wp-content/uploads/2017/10/small-example-pdf-file.pdf && $HOME/fuzzing_xpdf/install/bin/pdfinfo -box -meta $HOME/fuzzing_xpdf/pdf_examples/helloworld.pdf
```

## AFL

Remove the old build:

```shell
rm -r $HOME/fuzzing_xpdf/install && cd $HOME/fuzzing_xpdf/xpdf-3.02/ && make clean
```

Build Xpdf using the **afl-clang-fast** compiler:

```shell
export LLVM_CONFIG="llvm-config-11" && CC=$(which afl-clang-fast) CXX=$(which afl-clang-fast++) ./configure --prefix="$HOME/fuzzing_xpdf/install" && make && make install
```

Run AFL!

```shell
afl-fuzz -i $HOME/fuzzing_xpdf/pdf_examples/ -o $HOME/fuzzing_xpdf/out/ -s 123 -- $HOME/fuzzing_xpdf/install/bin/pdftotext @@ $HOME/fuzzing_xpdf/output
```

It took me about 7 mins to find 2 crashes:

![afl](./afl.png)

The payloads are saved in `$HOME/fuzzing_xpdf/out/default/crashes/`.

## Crash Analysis

Reproduce the crash:

```shell
$HOME/fuzzing_xpdf/install/bin/pdftotext "$HOME/fuzzing_xpdf/out/default/crashes/<your_filename>" $HOME/fuzzing_xpdf/output
```

The filenames will be different for each session. For me, it is:

```shell
$HOME/fuzzing_xpdf/install/bin/pdftotext "$HOME/fuzzing_xpdf/out/default/crashes/id:000000,sig:11,src:000349,time:236891,execs:136205,op:havoc,rep:16" $HOME/fuzzing_xpdf/output
```

and:

```shell
$HOME/fuzzing_xpdf/install/bin/pdftotext "$HOME/fuzzing_xpdf/out/default/crashes/id:000001,sig:11,src:000349,time:282507,execs:161195,op:havoc,rep:16" $HOME/fuzzing_xpdf/output
```

Digging deeper, we recompile the binary with debug info in order to investigate the crashes in GDB:

```shell
rm -r $HOME/fuzzing_xpdf/install && cd $HOME/fuzzing_xpdf/xpdf-3.02/ && make clean && CFLAGS="-g -O0" CXXFLAGS="-g -O0" ./configure --prefix="$HOME/fuzzing_xpdf/install/" && make && make install
```

Run GDB for crash 1:

```shell
gdb --args $HOME/fuzzing_xpdf/install/bin/pdftotext $HOME/fuzzing_xpdf/out/default/crashes/id:000000,sig:11,src:000349,time:236891,execs:136205,op:havoc,rep:16 $HOME/fuzzing_xpdf/output
```

Type `r` to run the program and hit the crash. Type `bt` to see the backtrace. Here we can see a lot of `Parser::getObj` gets invoked.

Run GDB for crash 2:

```shell
gdb --args $HOME/fuzzing_xpdf/install/bin/pdftotext $HOME/fuzzing_xpdf/out/default/crashes/id:000001,sig:11,src:000349,time:282507,execs:161195,op:havoc,rep:16 $HOME/fuzzing_xpdf/output
```

Same, type `r` and then type `bt`. Again, a lot of `Parser::getObj` gets invoked.

The CVE report of this bug can be found here:

https://www.cvedetails.com/cve/CVE-2019-13288/

It says:

> In Xpdf 4.01.01, the Parser::getObj() function in Parser.cc may cause infinite recursion via a crafted file. A remote attacker can leverage this for a DoS attack.

This report matches our observation from the GDB session.