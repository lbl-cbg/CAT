# CAT and BAT

## Introduction
Contig Annotation Tool (CAT) and Bin Annotation Tool (BAT) are pipelines for the taxonomic classification of long DNA sequences and metagenome assembled genomes (MAGs/bins) of both known and (highly) unknown microorganisms, as generated by contemporary metagenomics studies. The core algorithm of both programs involves gene calling, mapping of predicted ORFs against the NR protein database, and voting-based classification of the entire contig / MAG based on classification of the individual ORFs. CAT and BAT can be run from intermediate steps if files are formated appropriately (see Usage). A paper describing the algorithm is currently in the works.

## Dependencies and where to get them
Python 3 https://www.python.org/ (tested on version 3.5.2)

DIAMOND https://github.com/bbuchfink/diamond (tested on version 0.9.14)

Prodigal https://github.com/hyattpd/Prodigal (tested on version 2.6.3)

CAT and BAT have been thoroughly tested on Linux systems, and should run on Mac OS as well.

## Installation
No installation is required. You can run CAT and BAT by supplying the absolute path:

```
$ CAT_pack/CAT --help
```

Alternatively, if you add the files in the CAT\_pack directory to your $PATH variable, you can run CAT and BAT from anywhere:

```
$ CAT --version
```

CAT and BAT can also be installed via Bioconda, thanks to Silas Kieser:
```
$ conda install -c bioconda cat
```

## Getting started
To get started with CAT and BAT, you will have to get the database files on your system. You can either download preconstructed database files, or generate them yourself which will get you the latest versions of NR and the taxonomy files.

### Downloading the database files.
To download the database files, find the most recent version on tbb.bio.uu.nl/bastiaan/CAT\_prepare/, download and extract, and you are ready to go!
```
$ wget tbb.bio.uu.nl/bastiaan/CAT_prepare/CAT_prepare_20190108.tar.gz

$ tar -xvzf CAT_prepare_20190108.tar.gz
```

### Generating the database files yourself.
```
$ CAT prepare --fresh
```

This will download the taxonomy files from NCBI taxonomy to a taxonomy folder, and the NR database to a database folder. A DIAMOND database is constructed from the NR file. CAT prepare also generates a fastaid2LCAtaxid file, as the first accession numbers in the headers of NR are not necessarily the Last Common Ancestor (LCA) of all accession numbers in it. Moreover, the file taxids\_with\_multiple\_offspring is generated. CAT prepare will typically take a few hours to create a fresh database, and will use up to 100GB of memory.

If some of the files are already on your system (say the taxonomy files and the NR database) you can run:
```
$ CAT prepare --existing -d {folder containing NR} -t {folder containing taxonomy files}
```

CAT prepare will try to assess which files need to be downloaded and created and start from that point. CAT prepare only checks if the necessary files are there, not if they are correctly formatted.

### Running CAT and BAT.
The taxonomy folder and database folder created by CAT prepare are needed in subsequent CAT and BAT runs. They only need to be generated/downloaded once or whenever you want to update the NR database.

To run CAT on a contig set, each header in the contig fasta file (the part after '>' and before the first space) needs to be unique. To run BAT on set of MAGs, each header in a MAG needs to be unique within that MAG. If you are unsure if this is the case, you can just run CAT or BAT, as the appropriate error messages are generated if formatting is incorrect.

## Usage
After you have got the database files on your system, you can run CAT to annotate your contig set:

```
$ CAT contigs -c {contigs fasta} -d {database folder} -t {taxonomy folder}
```

Multiple output files and a log file will be generated. The final classification files will be called 'out.CAT.ORF2LCA.txt' and 'out.CAT.contig2classification.txt'.

Alternatively, if you already have a predicted proteins fasta file and/or an alignment table for example from previous runs, you can supply them to CAT, which will then skip the steps that have already been done start from there:

```
$ CAT contigs -c {contigs fasta} -d {database folder} -t {taxonomy folder} -p {predicted proteins fasta} -a {alignment file}
```

The headers in the predicted proteins fasta file must look like this '\>{contig}\_{ORFnumber}', so that CAT can couple contigs to ORFs. The alignment file must be tab-seperated, with queried ORF in the first column, NR protein accession number in the second, and bit-score in the 12th.

To run BAT on a set of MAGs:

```
$ CAT bins -b {bin folder} -d {database folder} -t {taxonomy folder}
```

Multiple output files and a log file will be generated. The final classification files will be called 'out.BAT.ORF2LCA.txt' and 'out.BAT.bin2classification.txt'.

Similarly to CAT, BAT can be run from intermidate steps if gene prediction and alignment have already been carried out once:

```
$ CAT bins -b {bin folder} -d {database folder} -t {taxonomy folder} -p {predicted proteins fasta} -a {alignment file}
```

## Interpreting output
The ORF2LCA output looks like this:

ORF | lineage | bit-score
--- | --- | ---
contig\_1\_ORF1 | 1;131567;2;1783272 | 574.7

Where the lineage is the full taxonomic lineage of the classification of the ORF, and the bit-score the top-hit bit-score that is assigned to the ORF for voting. The BAT ORF2LCA output file has an extra column where ORFs are linked to the MAG in which they are found.

The contig2classification and bin2classification output looks like this:

contig or bin | classification | number of ORFs on contig or in bin | number of ORFs classification is based on | lineage | lineage scores
--- | --- | --- | --- | --- | ---
contig\_1 | classified | 15 | 14 | 1;131567;2;1783272 | 1.00; 1.00; 1.00; 0.78
contig\_2 | classified (1/2) | 10 | 10 | 1;131567;2;1783272;1798711;1117;307596;307595;1890422;33071;1416614;1183438\* | 1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;0.23;0.23
contig\_2 | classified (2/2) | 10 | 10 | 1;131567;2;1783272;1798711;1117;307596;307595;1890422;33071;33072 | 1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;1.00;0.77

Where the lineage scores represent the fraction of bit-score support for each classification. **Contig\_2 has two classifications.** This can happen if the *f* parameter is chosen below 0.5. For an explanation of the **starred classification**, see 'Marking suggestive classifications with an asterisk' below.

To add names to the taxonomy id's in either output file, run:

```
$ CAT add_names -i {ORF2LCA / classification file} -o {output file} -t {taxonomy folder}
```

This will show you that for example contig\_1 is classified as Terrabacteria group. To only get official levels (*i.e.* superkingdom, phylum, ...):

```
$ CAT add_names -i {ORF2LCA / classification file} -o {output file} -t {taxonomy folder} --only_official
```

If you have named a CAT or BAT classification file with official names, you can get a summary of the classification, where total length and number of ORFs supporting a taxon are calculated for contigs, and the number of MAGs per encountered taxon for MAG classification:

```
$ CAT summarise -c {contigs fasta} -i {named CAT classification file} -o {output file}

$ CAT summarise -i {named BAT classification file} -o {output file}
```

CAT summarise currently does not support classification files wherein some contigs / MAGs have multiple classifications (as contig\_2 above).

## Marking suggestive classifications with an asterisk
When we want to confidently go down to the lowest taxonomic level possible for an classification, an important assumption is that on that level conflict between classifications could have arisen. Namely, if there were conflicting classifications, the algorithm would have made the classification more conservative by moving up a level. Since it did not, we can trust the low-level classification. However, it is not always possible for conflict to arise, because in some cases no other sequences from the clade are present in the database. This is true for example for the family Dehalococcoidaceae, which in our databases is the sole representative of the order Dehalococcoidales. Thus, here we cannot confidently state that an classification on the family level is more correct than an classification on the order level. For these cases, CAT and BAT mark the lineage with asterisks, starting from the lowest level classification up to the level where conflict could have arisen because the clade contains multiple taxa with database entries. The user is advised to examine starred taxa more carefully, for example by analysing sequence identity between predicted ORFs and hits, or move up the lineage to a confident classification (i.e. the first classification without an asterisk).

## Examples
Getting help for running the prepare utility:

```
$ CAT prepare --help
```

Create a fresh database, run CAT on a contig set with default parameter settings deploying 16 cores for DIAMOND alignment, name the contig classification output with official names, and create a summary:

```
$ CAT prepare --fresh -d CAT_database/ -t CAT_taxonomy/

$ CAT contigs -c contigs.fasta -d CAT_database/ -t CAT_taxonomy/ -n 16 --out_prefix first_CAT_run

$ CAT add_names -i first_CAT_run.contig2classification.txt -o first_CAT_run.contig2classification.official_names.txt -t CAT_taxonomy/

$ CAT summarise -c contigs.fasta -i first_CAT_run.contig2classification.official_names.txt -o CAT_first_run.summary.txt
```

Run the classification algorithm again with custom parameter settings and name the contig classification output with all names in the lineage:

```
$ CAT contigs --range 5 --fraction 0.1 -c contigs.fasta -d CAT_database/ -t CAT_taxonomy/ -p first_CAT_run.predicted_proteins.fasta -a first_CAT_run.alignment.diamond -o second_CAT_run

$ CAT add_names -i second_CAT_run.contig2classification.txt -o  second_CAT_run.contig2classification.names.txt -t CAT_taxonomy/
```

Run BAT on a set of MAGs with custom parameter settings and add names to the ORF2LCA output file, suppressing verbosity and not writing a log file:

```
$ CAT bins -r 10 -f 0.1 -b ../bins/ -s .fa -d CAT_database/ -t CAT_taxonomy/ -o BAT_run --quiet --no_log

$ CAT add_names -i BAT_run.ORF2LCA.txt -o BAT_run.ORF2LCA.names.txt -t CAT_taxonomy/
```
