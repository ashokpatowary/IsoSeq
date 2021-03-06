<h1 align="center"><img width="300px" src="doc/img/isoseq3.png"/></h1>
<h1 align="center">IsoSeq v3.0</h1>
<p align="center">Scalable De Novo Isoform Discovery</p>

***

*IsoSeq v3.0* contains the newest tools to identify transcripts in
PacBio single-molecule sequencing data.
Starting in SMRT Link v6.0.0, those tools power the
*IsoSeq GUI-based analysis* application.
A composable workflow of existing tools and algorithms, combined with
a new clustering technique, allows to process the ever-increasing yield of PacBio
machines with similar performance to *IsoSeq* versions 1 and 2.

## Availability
Latest version can be installed via bioconda package `isoseq3`.

Please refer to our [official pbbioconda page](https://github.com/PacificBiosciences/pbbioconda)
for information on Installation, Support, License, Copyright, and Disclaimer.

## Overview
 - [SMRTbell Designs](README_v3.0.md#smrtbell-designs)
 - [Workflow Overview](README_v3.0.md#workflow)
 - [Real-World Example](README_v3.0.md#real-world-example)
 - [FAQ](README.md#faq)

## Workflow

<img width="1000px" src="doc/img/isoseq3.0-workflow.png"/>

### Input
For each cell, the `<movie>.subreads.bam` and `<movie>.subreads.bam.pbi`
are needed for processing.

### Circular Consensus Sequence calling
Each sequencing run is processed by [*ccs*](https://github.com/PacificBiosciences/ccs)
to generate one representative circular consensus sequence (CCS) for each ZMW. Only ZMWs with
at least one full pass (at least once subread with SMRT adapter on both ends) are
used for the subsequent analysis. Polishing is not necessary
in this step and is by default deactivated through.
_ccs_ can be installed with `conda install pbccs`.

    ccs movie.subreads.bam ccs.bam --noPolish --minPasses 1

For **CCS version ≥ 4.0.0** use this call:

    $ ccs movie.subreads.bam ccs.bam --skip-polish --min-passes 1 --draft-mode winpoa --disable-heuristics

### Primer removal and demultiplexing
Removal of cDNA primers and identification of barcodes (if given) is performed using [*lima*](https://github.com/pacificbiosciences/barcoding),
which can be installed with `conda install lima` and offers a specialized `--isoseq` mode.

More information about how to name input primer(+barcode)
sequences in this [FAQ](https://github.com/pacificbiosciences/barcoding#how-can-i-demultiplex-isoseq-data).

    lima --isoseq --dump-clips ccs.bam primers.fasta demux.bam

The following is the `primer.fasta` for the Clontech SMARTer cDNA library prep, which is the officially recommended protocol:

    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGG
    >primer_3p
    GTACTCTGCGTTGATACCACTGCTT

The following are examples for barcoded samples using a 16bp barcode followed by Clontech primer:

    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGGG
    >brain_3p
    CGCACTCTGATATGTGGTACTCTGCGTTGATACCACTGCTT
    >liver_3p
    CTCACAGTCTGTGTGTGTACTCTGCGTTGATACCACTGCTT

*lima* will remove unwanted combinations and orient sequences to 5' -> 3' orientation.

From here on, execute the following steps for each output BAM file.

### Clustering and polishing
*IsoSeq v3* wraps all tools into one fat binary.

    $ isoseq3
    isoseq3 - De Novo Transcript Reconstruction

    Tools:
        cluster   - Cluster CCS reads to transcripts
        polish    - Polish the clustering output
        summarize - Create a barcode overview CSV file

    Examples:
        isoseq3 cluster movie.consensusreadset.xml unpolished.bam
        isoseq3 polish unpolished.bam movie.subreadset.xml polished.bam
        isoseq3 summarize polished.bam summary.csv

#### Clustering and transcript clean up
Compared to previous IsoSeq approaches, *IsoSeq v3* performs a single clustering
technique.
Due to the nature of the algorithm, it can't be efficiently parallelized. It is advised to give this step as many cores
as possible. The individual steps of *cluster* are as following:
 - [Trimming](https://github.com/PacificBiosciences/trim_isoseq_polyA) of polyA tails `--require-polya`
 - Rapid concatmer [identification](https://github.com/jeffdaily/parasail) and removal
 - Clustering using hierarchical n*log(n) [alignment](https://github.com/lh3/minimap2) and iterative cluster merging
 - Unpolished [POA](https://github.com/rvaser/spoa) sequence generation

##### Input
The input file for *cluster* is one demultiplexed CCS file:
 - `<demux.ccs.bam>` or `<demux.ccs.consensusreadset.xml>`

##### Output
The following output files of *cluster* contain unpolished isoforms:
 - `<prefix>.bam`
 - `<prefix>.flnc.bam`
 - `<prefix>.fasta`
 - `<prefix>.bam.pbi` <- Only generated with `--pbi`
 - `<prefix>.transcriptset.xml` <- Only relevant for pbsmrtpipe
 - `<prefix>.consensusreadset.xml` <- Only relevant for pbsmrtpipe

Example invocation:

    isoseq3 cluster demux.P5--P3.bam unpolished.bam -j 32 [--split-bam 24]

#### Polishing
The algorithm behind *polish* is the *arrow* model that also used for CCS
generation and polishing of de-novo assemblies. This step can be massively
parallelized by splitting the `unpolished.bam` file. Split BAM files can be
generated by *cluster*.

##### Input
The input files for *polish* are:
 - `<unpolished>.bam` or `<unpolished>.transcriptset.xml`
 - `<movie_name>.subreads.bam` or `<movie_name>.subreadset.xml`

##### Output
The following output files of *polish* contain polished isoforms:
 - `<prefix>.bam`
 - `<prefix>.bam.pbi` <- Only generated with `--pbi`
 - `<prefix>.transcriptset.xml`
 - `<prefix>.hq.fasta.gz` with predicted accuracy ≥ 0.99
 - `<prefix>.lq.fasta.gz` with predicted accuracy < 0.99
 - `<prefix>.hq.fastq.gz` with predicted accuracy ≥ 0.99
 - `<prefix>.lq.fastq.gz` with predicted accuracy < 0.99

Example invocation:

    isoseq3 polish unpolished.bam m54020_171110_2301211.subreads.bam polished.bam



## Real-world example
This is an example of an end-to-end cmd-line-only workflow to get from
subreads to polished isoforms.

    $ wget https://downloads.pacbcloud.com/public/dataset/RC0_1cell_2017/m54086_170204_081430.subreads.bam
    $ wget https://downloads.pacbcloud.com/public/dataset/RC0_1cell_2017/m54086_170204_081430.subreads.bam.pbi

    $ ccs --version
    ccs 3.1.0 (commit v3.1.0)

    $ time ccs m54086_170204_081430.subreads.bam m54086_170204_081430.ccs.bam \
               --noPolish --minPasses 1

    real    50m43.090s
    user    3531m35.620s
    sys     24m36.884s

    $ cat primers.fasta
    >primer_5p
    AAGCAGTGGTATCAACGCAGAGTACATGGGG
    >primer_3p
    AAGCAGTGGTATCAACGCAGAGTAC

    $ lima --version
    lima 1.7.1 (commit v1.7.1)

    $ time lima m54086_170204_081430.ccs.bam primers.fasta demux.bam \
                --isoseq --dump-clips

    real    0m6.543s
    user    0m51.170s

    $ ls demux*
    demux.json  demux.lima.counts  demux.lima.report  demux.lima.summary  demux.primer_5p--primer_3p.bam  demux.primer_5p--primer_3p.subreadset.xml

    $ time isoseq3 cluster demux.primer_5p--primer_3p.bam unpolished.bam --verbose
    Read BAM                 : (200740) 8s 313ms
    India                    : (197869) 9s 204ms
    Save flnc file           : 35s 366ms
    Convert to reads         : 36s 967ms
    Sort Reads               : 69ms 756us
    Aligning Linear          : 42s 620ms
    Read to clusters         : 7s 506ms
    Aligning Linear          : 37s 595ms
    Merge by mapping         : 37s 645ms
    Consensus                : 1m 47s
    Merge by mapping         : 8s 861ms
    Consensus                : 12s 633ms
    Write output             : 3s 265ms
    Complete run time        : 5m 12s

    real    5m12.888s
    user    58m35.243s

    $ ls unpolished*
    unpolished.bam  unpolished.bam.pbi  unpolished.cluster  unpolished.fasta  unpolished.flnc.bam  unpolished.flnc.bam.pbi  unpolished.flnc.consensusreadset.xml  unpolished.transcriptset.xml

    $ time isoseq3 polish unpolished.bam m54086_170204_081430.subreads.bam polished.bam --verbose
    14561

    real    60m37.564s
    user    2832m8.382s
    $ ls polished*
    polished.bam  polished.bam.pbi  polished.hq.fasta.gz  polished.hq.fastq.gz  polished.lq.fasta.gz  polished.lq.fastq.gz  polished.transcriptset.xml

If you have multiple cells, you should run `--split-bam` in the cluster step which will produce chunked cluster results. Each chunked cluster result can be run as a parallel polish job and merged at the end. The following example splits into 24 chunks. `sample.subreadset.xml` is the dataset containing all the input cells. The `isoseq3 polish` jobs can be run in parallel.

    $ isoseq3 cluster demux.primer_5p--primer_3p.bam unpolished.bam --split-bam 24
    $ isoseq3 polish unpolished.0.bam sample.subreadset.xml polished.0.bam
    $ isoseq3 polish unpolished.1.bam sample.subreadset.xml polished.1.bam
    $ ...

## DISCLAIMER

THIS WEBSITE AND CONTENT AND ALL SITE-RELATED SERVICES, INCLUDING ANY DATA, ARE PROVIDED "AS IS," WITH ALL FAULTS, WITH NO REPRESENTATIONS OR WARRANTIES OF ANY KIND, EITHER EXPRESS OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, ANY WARRANTIES OF MERCHANTABILITY, SATISFACTORY QUALITY, NON-INFRINGEMENT OR FITNESS FOR A PARTICULAR PURPOSE. YOU ASSUME TOTAL RESPONSIBILITY AND RISK FOR YOUR USE OF THIS SITE, ALL SITE-RELATED SERVICES, AND ANY THIRD PARTY WEBSITES OR APPLICATIONS. NO ORAL OR WRITTEN INFORMATION OR ADVICE SHALL CREATE A WARRANTY OF ANY KIND. ANY REFERENCES TO SPECIFIC PRODUCTS OR SERVICES ON THE WEBSITES DO NOT CONSTITUTE OR IMPLY A RECOMMENDATION OR ENDORSEMENT BY PACIFIC BIOSCIENCES.
