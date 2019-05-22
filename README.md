# Cache miss rate sequence dataset

This data set was used in the experiments described in "Cache Miss Rate Predictability via Neural Networks",
by Rishikesh Jha, Arjun Karuvally, Saket Tiwair, and Eliot Moss, College of Information and Computer Sciences,
University of Massachusetts Amherst, 140 Governors Dr, Amherst MA 01002.
The paper appeared at the Workshop on ML for Systems at NeurIPS 2018, December, 2018.
You can email Prof. Moss at moss@cs.umass.edu.

# How to clone the repository

The files in this repository are quite large, we we use git's Large File Storage (LFS) extension.
Before cloning, you need to have git-lfs installed on your system.  Then, to do the actual cloning,
use the command `git lfs clone <address of the repo>` where the address can be seen from the Clone
or Download button of the repository in GitHub.  (There is some evidence that on some systems you
can use `git clone` (without the `lfs`), and that may even be preferred, but also, depending on
your system, that may just not work.)  Note that cloning will take a while since the size
of the repository is 27+ Gb.  And of course you will need plenty of space (space use will be triple
the 27 Gb!).  You can delete the .git subtree of the cloned repo when done, to reclaim 2/3 of the space
(we do not anticipate incremental changes to the repo, except perhaps the README and such).

# What this GitHub directory contains

In addition to this README, this folder contains a collection of zip files, Analyz01.zip through Analyz15.zip.
Originally they formed one large zip file, but GitHub disallows files larger than 2 Gb, so that large one is
split into portions.  You may wish to join the contents together into a single zip file after downloading,
since the splitting is somewhat arbitrary.  To do this, unzip each of this .zip files (perhaps into a clean
working directory you create for the purpose), then zip the extract files together again.  The files in the
.zip archives are .gz files, so there is no real point to compressing the .zip archive -- just use -0
compression.

# What the zip archives contain

The archives contain 2052 .gz files, each one containing an analysis (there are 18 analyses,
described in a moment) for a given program run (thus there are 114 program runs).  It is perhaps
easier to understand these by following the process by which these files were built.

1.  A program is run using *valgrind* to create a trace of every memory reference the program
makes.  Specifically, we used valgrind's *lackey* tool.  This "raw" trace includes instruction
fetches, data reads, data writes, and data read-modify-write accesses made by every instruction.
Each access includes the virtual memory address and the number of bytes accessed.  We did this
on an x86 processor.  We regret that the details of which compilers were used, etc., are somewhat
lost in the mists of time at this point.  However, C and Fortran programs were compiled to a high
optimization level, etc.

1.  We choose a *cache line size* and process the trace according to that line size.  For our work we
used two line size, 64 and 4096.  Line size 64 is typical for Intel processors and corresponds to the
usual L1, L2, etc., caches.  Line size 4096 captures the idea of referring to *pages*, and this can be
used to analyze TLB (translation lookaside buffer) and paging behavior.  (In practice, these benachmarks
are designed not to page, so these traces probably are not well-suited to page fault analyses.)

1. We further choose whether we are processing *instruction* accesses (only), *data* accesses (only),
or *both*.  Each of these has it use, depending on the level of cache one considers, etc.  We abbreviate
this choice to *i*, *d*, or *b*.

1. Given the cache line size and the desired access kinds (i, d, b), we map the sequence of accesses from
virtual addresses to the cache lines referenced (divide by the line size to get a line number).  However,
if an access crosses a line boundary, it will be treated as accessing the lower line number then the higher
one.  (Typically this happens only with instruction accesses.)

1. We applied Belady's LRU Stack algorithm to the sequence of line numbers accessed to determine a
corresponding sequence of stack depths.  Note that if an access has stack depth *k*, then it will miss
in a perfect LRU cache of size 1 through *k*, but hit in a cache of size more than *k*.  (We start our
depth numbers at 0, if this seems off by one to you!)

1. We *group* these depth numbers into groups of 100,000 instructions.  (Note that we can tell instructions
by instruction accesses.)  We then construct of *histogram* that indicates, for each depth, how many references
had that depth number, within this window.  One can then determine the number of hits and misses with that window,
for *any* chosen cache size.

1. We process these histograms and for each of them we output a line having this information:
  1. Number of accesses in that window
  1. Minimum cache size needed to achieve a miss rate no great than: 10%, 6%, 4%, 2%, 1%, and 0.5%
  1. For cache sizes 4K, 8K, 16K, 32K, 64K, 128K, 256K, 512K, 1M, 2M, 4M, 8M, and 18M bytes, the number of misses
  for that cache size, and the miss rate for that cache size.  For the paper we used only the 32K byte cache
  size data.
The file has a header line that indicates these miss rates and cache sizes (in between the cache size in bytes
is the cache size divided by 64, which is the number of lines in the 64 byte line size case).

# The program runs

There are two broad categories of runs, namely Java programs taken from the DaCapo benchmark suite, and C/Fortran programs
taken from the SPEC cpu2000 benchmark suite.  The Java programs were run on three different Java virtual machines (JVMs),
JikesRVM, HotSpot, and IBM's J9.  Programs have varying numbers of different inputs.  In one case we ran the same (Java) program
on the same input multiple times, which can be used to examine the extent of variation from run to run.  (The SPEC benchmarks
should not vary.)

# File naming scheme

A given .gz file name has this form:

*program*-*input*-*detail*-*A*-l*LL*-p4096-w100000i.analyzed-*NN*.gz

For Java programs, *detail* will be *JikesRVM*, *HotSpot*, or *J9*, indicating the JVM.  For SPEC programs, *detail* will be the program name again, but possibly with a numeric suffix.  This is because some SPEC benchmarks, such as *gcc*, run their program more than once.  For DaCapo Java programs, *input* will generally be *tiny*, *small*, *default*, *large*, or *huge*, as in DaCapo.  If there is a *-n* added, that indicates a specific run of a program run multiple times, notably *pmd-small*.  For SPEC, the *input* will be *ref*, i.e., the reference (longest running) input.

The *A* will be *i*, *d*, or *b*, the kind of accesses included in the analysis.

*LL* will be 64 or 4096, the cache line size used in the analysis.

*NN* indicates how many histograms are grouped together for each line of output.  Thus, 1 indicates groups of 100,000 instructions, 10 indicates groups of 1,000,000 instructions, and 100 indicates groups of 10,000,000 (and the files will be correspondingly smaller, having that many fewer lines).

The p4096 is just an historical artifact and has no real meaning at this point.  The w100000i indicates that the original window size for developing the base histograms is 100,000 instructions.
