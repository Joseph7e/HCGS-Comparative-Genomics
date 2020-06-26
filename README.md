# HCGS-Comprative-Genomics
NCBI download and Orthofinder analysis



# MDIBL-T3-WGS-Comparative

## Our Starting data

```bash
ls /home/share/workshop/faa_files/
```

## What we will be doing

We will be using **Orthofinder** for our main comparative genomic analysis. The manual is very detailed so take some time to readit. To run the program we will need some genomes to compare.

Orhtofinder Manual: https://github.com/davidemms/OrthoFinder

The program takes a set of protein sequences for each species and runs pair-wise comarisons to identify orhtologous groups. For each orthogroup a gene tree is calculated and in the end an overall species tree is computed. To get any sort of meaningful phylogenetic tree we need to be sure to include **at least four different genome datasets**. Ideally we would run this program with all of the avaialble sequences on NCBI. As you can imagine, a pairwise comparison with 1,185 Streptomyces genomes will take a lone time (days). We will therfore run it with a reduced set. Next we will determine what genomes we want to download and go over the best ways to retrieve them from NCBI.


## Set up working directories
```bash
cd ~/genomics_tutorial/
mkdir genbank_downloads
cd genbank_downloads/
```

## Locate Reference Data on NCBI

FAQs about genome download from NCBI: https://www.ncbi.nlm.nih.gov/genome/doc/ftpfaq/#GBorRS

Whatever method you use be sure to grab an outgroup, or don't thats your call.

### Method 1: Download speicifc genomes

We ran a NCBI blast during the genome assembly tutorial. This BLAST should have given you the closest match against the nt database. Chances are it 'hit' well to many genomes. Choose the top hit to a full genome and follow the links to retriev the download link of the genome FAA from the ftp site.

Alternatively, you can download the reference genome used as part of the MR study in staphylococcus.

Staphylococcus aureus ATCC 29213 is the reference strain.
https://www.ncbi.nlm.nih.gov/assembly/GCF_001879295.1

FOllow the links to the FTP download.

```bash
wget "https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/001/879/295/GCF_001879295.1_StAu00v1/GCF_001879295.1_StAu00v1_protein.faa.gz"
```


### Method 2: Download all refseq genomes for your genus

link to NCBI prokaryote tables: https://www.ncbi.nlm.nih.gov/genome/browse#!/prokaryotes/
link to genome reports FTP: ftp://ftp.ncbi.nlm.nih.gov/genomes/GENOME_REPORTS/

* Download the genome report file for all all of prokaryotes

We will download the file directly to the server, there is no need to download it to your computer. Right lick on the link and copy the link address. This is a big file so we will filter it a bit first before opening it with tabview.

This file has a lot of useful information. For now we really care about column 21 which is the downloa link for the genome on the ftp site. Copy that link and paste it into a browser to see the files. We will then download the FAA files to the server.


```bash
# download the file
wget "ftp://ftp.ncbi.nlm.nih.gov/genomes/GENOME_REPORTS/prokaryotes.txt"

# view it
tabview prokaryotes.txt

# grep for species in question and view.
grep -i "Staphylococcus" prokaryotes.txt | grep REFR | tabview -

# print the download commands
grep "Staphylococcus" prokaryotes.txt | grep REFR | awk -F'\t' '{print "wget "$21"/*protein.faa.gz"}'

# download all the faa files automatically.
grep "Staphylococcus" prokaryotes.txt | grep REFR | awk -F'\t' '{print $21"/*protein.faa.gz"}' | xargs -P16 wget -i

# or even better, rename the files as you go
grep "Staphylococcus" prokaryotes.txt | grep REFR | sed 's/ /_/g' | awk -F'\t' '{print $1"_"$19".faa.gz",$21"/*protein.faa.gz"}' | xargs -n 2 -P 1 wget -O
```


Remove the empty files, some of them don't have annotations, so lets remove them.

```bash
zgrep -c '>' *.faa.gz
zgrep -c '>' *.faa.gz | awk -F':' '$2 == 0'

# automatic deletion with xargs
zgrep -c '>' *.faa.gz| awk -F':' '$2 == 0 {print $1}' | xargs rm
```

Unzip all the files

```bash
gunzip *.faa.gz
```


## Set up orthofinder directory

```bash
# move to analysis folder
mkdir ~/genomics_tutorial/orthofinder-analysis
cd ~/genomics_tutorial/orthofinder-analysis


# create a soft link to the FAA files we just downloaded
ln -s ../genbank_downloads/*.faa ./

# create a soft link to the FAA fles from our PROKKA analysis
ln -s /home/share/workshop/faa_files/*.faa ./
```

## Count the number of proteins in all the starting files
Think about what these numbers tell us right off the bat.

```bash
grep -c '>' *.faa
```

## Run Orthofinder2

The input to the program is a directory containing a FAA file for each species.

```bash
# view the manual
orthofinder2 --help
# run the program, it will take some time
nohup time orthofinder2 -t 16 -a 16 -S diamond -f ./ &
```

## Examine the output files

I will review some, but not all of the files. The manual goes into extensive detail.

```bash
cd Results*/
ls
```

### * Orthogroups.csv

A **tab** seperated table. Each orthogroup is a raw, each column is a different sample.

The table provides all of the data for orthogroups that are in at least two different samples. If a sample has more than one protein for that particular orthogroup than it will have a comma seperated list for the entry. 

```bash
tabview Orthogroups.csv
```

### * Orthogroups_UnassignedGenes.csv

The same style table. Instead this one contains Orthogroups that are not belonging to an orthogroup, they are unique to a single sample. As you scroll down you should notice the proteins belong to different samples.

```bash
tabview Orthogroups_UnassignedGenes.csv
```

###  * Orthogroups.GeneCount.csv

My favorite 'Orthogroup' Output file. Orthogroups are the rows, columns are gene counts per species. This can be easily parsed to see what orthogroups are specific to waht species. It provides total gene counts for each sample.

```bash
tabview Orthogroups.GeneCount.csv
```

* add annotations from a reference sequence

~/orthogroups_add_annotations.py <reference_faa> Orthogroups.txt  Orthogroups.GeneCount.csv

```bash
orthogroups_add_annotations.py ../GCF_000203835.1_ASM20383v1_protein.faa Orthogroups.txt  Orthogroups.GeneCount.csv | tabview -
```


## Statistics

### * Statistics_Overall.csv

A file containing the overall statistcis for the analysis. Total number of genes in the dataset etc. 

```bash
tabview Statistics_Overall.csv
```

### * Statistics_PerSpecies.csv

In my opinion this is the most important statistics output file. It provides details for each sample. How many genes were speciifc to that sample. If you want to know a quick statistics of how 'differen't your genome is, this is it.

```bash
tabview Statistics_PerSpecies.csv
```

### * WorkingDirectory/

All of the work that external programs like BLAST or DIAMOND. 'ls' this directory. It contains all the results for each pairwise comparison.

### * Orthologues_DATE/

This directory contains a lot of useful data related to the Orthofinder analysis and how they commpute the phylogenetic trees.

#### * Recon_Gene_Trees/

A directory containing inferred trees for every orthogroup.

#### * SpeciesTree_rooted.txt

A rooted-species tree. Orthofinder commputes a root for the tree automatically. You can view this in any tree viewing program like FigTree or TreeView (macs). This file is in newick format. Check it out.

```bash
more Orthologues*/SpeciesTree_rooted.txt
```

## Export the tree file and view.


## Bonus - Figures
```
orthogroups_add_annotations.py ../GCF_000203835.1_ASM20383v1_protein.faa Orthogroups.txt  Orthogroups.GeneCount.csv

orthotools-venn.py Results_*/ PROKKA_*.faa species1.faa species2.faa  venn

orthotools-UpSet.R Results_*/Orthogroups.GeneCount.csv

```
