# notes on STRIPE-seq processing workflows

## history

STRIPE-seq processing has taken a couple different forms so far, neither of which I found particularly adequate in terms of speed or ease of use.

**GoSTRIPES** is [the original](https://github.com/BrendelGroup/GoSTRIPES) `make`-based workflow by the Brendel group circa 2019.  It provides a template Makefile, scripts, and eventually a Singularity container to hold all dependencies.  The user must manually set paths for their raw R1 and R2 read files, and there isn't a convenient way provided to automate the workflow for many samples.

It proceeds as follows:

1. `fastqc` on the raw reads (must specify threads in the Makefile, runs `gunzip` if required)
2. `trimmomatic` to trim Illumina adapters (must specify threads and TrueSeq2 fasta path, not logged)
3. `tagdust` to remove rRNA contaminant reads (must specify threads and rRNA file)
4. TSO adapter removal by using custom scripts, `umi_tools`, and `cutadapt`: 
  - `fastq-interleave` shell script to merge R1 and R2 to a temp file
  - `selectReadsByPattern.pl` filters interleaved temp file by PATTERN (UMI [NNNNNNNN] + LINKER [TATAGGG]) (writes log file, and apparently de-interleaves the selected reads as well)
  - `umi_tools` to extract UMI from reads and add to read name (must specify UMI, single threaded, no logging)
  - `cutadapt` to remove the remaining LINKER from the selected, UMI-extracted reads (must specify threads, min read length, linker.  writes log file)
 5. Another round of adapter removal and selection to excise the remainder of the TSO
   - `cutdapt` to remove CCTACACGACGCTCTTCCGATCT (non-UMI, non-linker, P5-derived) from the front, allowing for overlap and errors (must provide threads, writes log file)
   - `selectAdapterTrimmedReads.pl` compares the the trimmed and untrimmed fastqs and writes interleaved trimmed and untrimmedout output (single threaded, not sure why this is necessary except for comparative analysis)
   - `selectReadsByPattern.pl` again to pick out the PATTERN reads (and interleave)
   - `umi_tools` again to exptract UMIs from selected reads and add to their readnames
   - `fastq-interleave` to recombine R1 and R2
   - a rule to combine the the interleaved secondary trimmed reads with some untrimmed(?) reads, and then `fastq-deinterleave` them (the Makefile is getting really obscure to me at this point - I can't where an 'unt_*.fastq' file is ever written )
   - (I'm assuming this entire second round is necessary to deal with P5-containing reads)
6. Another `fastqc` run on the trimmed and cleaned reads
  - (I note there's also never any fastqc_data.txt ever written anywhere, so this rule will always run.  I'm assuming this is intentional?)
7. Printing some basic statistics on the raw vs. cleaned reads with `sstats-pe.sh` custom shell script (not logged)
8. Alignment via `STAR`, including generating the index (must provide threads/paths, automatically logged)
9. Alignment processing with `samtools`
  - uses `samtools` to filter flags for proper pairs (include 2, exclude 256)
  - `samtools sort` to sort reads by name (set to use a ton of memory, must set threads), piped to `samtools fixmate` to remove unmapped and secondary alignments, resorted, then sent to `samtools markdup` to remove PCR duplicates (nothing logged)
  - generates a SAM file from the dedup'd bam via `samtools sort`, sorted by name
  - custom script `scrubSAMfile.pl` to select for reads that are paired, have no greater than 3 5' soft clipped bases, limit intron length (50-5000bp), and filters TLEN (less <= 1500) (these seem awfully large for yeast data, writes a log file)
10. Converts the filtered SAM file back to BAM
  - saves the SAM header via `samtools view`
  - `cat`s the original SAM header to the filtered SAM, `samtools sort`s, and `samtools view` to make BAM with header
11. Cleans up the output dir for intermediate files, moves final output and log files to new separate dirs
12. Final clean up to remove the scratch dir, zip the final output store dir, and delete it (is it deleting what it bothered to zip??)

I find this appraoch pretty convoluted and hard to maintain.  At each step, intermediate files are written to disk for the next round which takes I/O time, and sometimes there is a log written but not always.  There are also single-threaded bottle necks at `umi_tools` and the custom perl and shell scripts, which, if we must use them, could be parallelized.  `make` does have parallelization features, but I'd rather not stick with this general approach.


*gostripesR* is [the next iteration]()
