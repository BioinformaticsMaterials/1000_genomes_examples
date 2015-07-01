# 1000_genomes_examples

The 1000 Genomes project (http://www.1000genomes.org/) is an amazing public
resource coordinated by the NIH. It's a "deep catalog of human genetic
variation" for more than 2500 people, all carefully curated, organized and
published on the web.

Because of the size and complexity of the data, and because of the significant
interest in understanding human genetics, these data are often used as examples
of "big data" analysis. A typical example of this kind of thing can be found
in the Google genomics project https://github.com/googlegenomics.

## That's cool and all, but...

my personal opinion is that folks get too worked up over "big data" solutions
and sometimes too focused on distributed system implementation details (Spark,
etc.) when simple tools like R work just fine.

I am very impressed by a recent article by Michael Isard, Frank McSherry and
Derek Murray (summarized 
http://www.frankmcsherry.org/graph/scalability/cost/2015/01/15/COST.html).  In
that article, McSherry and his coauthors show that careful thinking about some
network analysis problems let them run better on a laptop than on a 128 CPU
core cluster using a few common "big data" computing frameworks.

This note presents a few simple but useful examples using the 1000 genomes data
in the spirit of Isard, McSherry and Murray. The examples are often used to
showcase big data analysis frameworks. This note shows how they can sometimes
be easily and efficiently solved on a decent laptop.

## Variant data

The examples use "variant call format" (VCF) files following an NCBI-specific
format available from links shown in the code snippets below. Loosely,
"variants" are places on the genome that vary from a reference genome in a
cataloged way. Variants include single-nucleotide polymorphisms (a.k.a. SNPs,
basically a single base change along the genome) and larger "structural"
alterations. The 1000 genomes project catalogs about 81 million variants.

Variant data are stored in files by chromosome. Each file contains a set of
header lines beginning with the `#` character, followed by one line per
variant. 
Full file format details are described in
www.1000genomes.org/wiki/analysis/variant%20call%20format/vcf-variant-call-format-version-41.

Variant data files for each chromosome are available from
ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/release/20130502/.

Discounting the header lines, a variant line in the data files consists of some
information columns about the variant followed by one column for each sample
(person) that indicates if they exhibit the variant on either or both
chromosomes. For example, part of a typical line (showing only the first 5
columns and 10-15 columns) looks like:

```
zcat ALL.chr20.phase3_shapeit2_mvncall_integrated_v5.20130502.genotypes.vcf.gz | sed /^#/d | cut -f "1-5,10-15" | head -n 1

20      60343   .       G       A   ...    0|0     0|0     0|0     0|0     0|0     0|0
```
This variant is on chromosome 20 at position 60343. The reference nucleotide is G and
the variant is A. Note that in some cases, more than one possibility may be listed
in which case the variants are numbered 1,2,3,...
Columns 10-15 show that none of the first 6 people in the database
have this variant on either strand of DNA `0|0`. Someone with the G to A variant
on the 2nd strand of DNA will display `0|1`, for example.

Numerous full-featured VCF file parsers exist for R, see for example 
the http://bioconductor.org project. But the simple
analyses considered in this project don't need to read VCF files in full
generality, and we can also benefit from the knowledge that the 1000 genomes
project follows a somewhat restricted VCF subset. I wrote the really simple
32-line C parser program
https://github.com/bwlewis/1000_genomes_examples/blob/master/parse.c to take
advantage of these observations and load a subset of VCF data from the 1000
genomes project into R pretty quickly.

The simple parser program turns VCF files into comma-separated output with four
or three columns: variant number (just the line offset in file), sample number
(person), alternate number on first strand or haploid alternate, optional
alternate number on 2nd strand (diploid). For example chromosome 20 again:
```
cc parse.c
zcat ALL.chr20.phase3_shapeit2_mvncall_integrated_v5.20130502.genotypes.vcf.gz  | sed /^#/d  | cut  -f "10-" | ./a.out | head

1,898,0,1
2,1454,0,1
3,462,0,1
3,463,1,0
```
The output says that person 898 has variant number 1 (the first listed) for
chromosome 20 present on the 2nd strand of DNA. ANd person 1454 has variant
number 2 present on the 2nd strand of DNA, and so on.

For our purposes in the following examples, this simple C parser quickly
converts the 1000 genomes VCF data into a variant number by person number
table.

The 1000 genomes project also includes ethnicity and some other phenotypic data
about each of the more than 2500 people whose genomes are represented.
Those data are available from:
ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/working/20130606_sample_info/20130606_g1k.ped
and are easy to directly read into R.
