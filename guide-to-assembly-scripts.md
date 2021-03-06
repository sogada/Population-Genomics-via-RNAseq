# Transcriptome assembly in the Palumbi Lab

## GENERAL INFORMATION
- Scripts are formatted for Stanford internal use. We use SLURM to send jobs to Stanford's Sherlock cluster.

### How to create a Sherlock account 
- to get access and support, ask PI to e-mail and give you permission research-computing-support[at]stanford.edu
- download [Kerberos](https://uit.stanford.edu/service/kerberos)
- [Sherlock Wiki] (http://sherlock.stanford.edu/mediawiki/index.php/Main_Page)
- To configure your SSH client to pass those Kerberos credentials for MAC, open terminal (Click Spotlight, type in terminal) and run the following commands:
```
	mkdir -p ~/.ssh 
	echo "Host sherlock sherlock.stanford.edu sherlock* sherlock*.stanford.edu   
	GSSAPIDelegateCredentials yes   
	GSSAPIAuthentication yes" >> ~/.ssh/config 
```

- create project directory in `$PI_SCRATCH/` (1TB space available)
	- more space and accessible to other users in the lab 
- your personal directory `$HOME` is 15GB
	- you can also use this space for your research, but it is not easily accessible to other lab members
- [more info on Sherlock storage](http://sherlock.stanford.edu/mediawiki/index.php/DataStorage)

### Downloading & storing data from Gnomex
- go to the directory where you want your raq sequence files
- `wget http://monalisa.cern.ch/FDT/` 
- use command line from Gnomex to download files to Sherlock
- backup raw files on external harddrive in lab
- can also backup to unlimited space with your Stanford account and google drive
	- [Sherlock Gdrive tutorial](http://sherlock.stanford.edu/mediawiki/index.php/DataStorage) 

### Sherlock basics 

- to log in:
	```
	kinit user@stanford.edu #& type in pw to gain permission
	ssh user@sherlock.stanford.edu #to access cluster
	```

- to move around:
`cd $HOME` is your personal directory
	 - limited to 15GB storage
	 - not accessible to other users

- all common use scripts: `cd $PI_HOME/scripts`
- all common use programs: `cd $PI_HOME/programs`

- to add a program to Sherlock
	
	```
	cd $PI_HOME/programs
	wget <linktofiledownload>
	unzip <file> 
	#or gzip <file.gz>
	```

- modifying your path to include scripts & programs:
	- this allows you to access these directories from any directory without having to hardcode the path	

	 
	 ```
	 cd ~
	 ls -a #list hidden items
	 nano .bashrc
	 #paste the following into your .bashrc
	 export PATH="$PI_HOME/programs:$PATH"
	 export PATH="$PI_HOME/scripts:$PATH"
	 export PATH="/share/PI/spalumbi/programs/anaconda/bin:$PATH"
	 ```
- ways to submit jobs:
	- `login_node` this is the default, it's for small jobs and moving around sherlock
	- `sdev` is for running small test jobs straight from the command line
		- one hour time limit
		- use `sdev -t 2:00:00` to get max 2 hrs
	- `bash` is for  use for small jobs in terminal
	- `sbatch` submits the job generated by your script to the cluster

- to run your job on different nodes, in your batch script, add: 
	- `#SBATCH -p owners` access to 600 nodes only available to owners. You will be kicked off an owner node if that owner logs on. Always check your slurm file to see if your job was aborted
	- `#SBATCH -p spalumbi` 256GB memory node
	- `#SBATCH -p hns` 1TB memory node
	- `#SBATCH -p spalumbi,hns` #will submit your job to whatever node is available first

- to check on status of job
	- `squeue -u username`
	- `squeue | grep 'spalumbi` #to see our nodes usage

- to cancel a job
	- `scancel <jobID>` to cancel 1 job
	- `scancel -u <username>` will cancel all jobs

- to see how much memory your job has used so far
	- ` sstat --format JobID,NTasks,nodelist,MaxRSS,MaxVMSize,AveRSS,AveVMSize $JOBNUMBER`

### Best practices:

Back up your scratch directory! 

- You can use the Sherlock data transfer node to move large datasets onto a backup hard drive


```
kinit username@stanford.edu
rsync -avz --progress --stats -e 'ssh -o GSSAPIAuthentication=yes' user@sherlock-dtn.stanford.edu:/<sherlock directory> <backup location>
```

- You can also use your Stanford Google Drive (with unlimited storage) to back up your files


```
ml load gdrive
gdrive -help
gdrive upload --recursive <path>
```
- once you're linked to your google account, there is also an example gdrive-upload.sh script in the shared scripts directory

- Check to see how much space you're taking up in the shared SCRATCH directory
	- `du -sh * | sort -h`

- `chmod 775 *` programs, scripts, some files that you create so others can use them

- `chmod 444` files you don’t want to accidentally write over, 
	`chmod -R ###` for directories

- if downloading a program for the first time, move the executable of the program to the programs folders
 `mv <program> $PI_HOME/programs`

- if downloading a new version of a program, rename with version to not override any executables in that directory

- try to create scripts that are not hard-coded 

- comment your scripts!

- Tips for checking outputs along the way:

	- always look at your slurm files for errors
		- `cat slurm*`
	- when making scripts that split up data into TEMP files for parallel processing, add a line that states your input file in the slurm output
		- `echo $1` if $1 is your input file
	- to count lines in a file
		- `wc - l <filename>`
	- to count contigs in a file
		- `grep -c “>” <filename>`



## DE NOVO TRANSCRIPTOME ASSEMBLY

### 1) Trim & Clip (Trimmomatic) 
- this program removes adapters, low quality seqs, etc.
- [webpage](http://www.usadellab.org/cms/?page=trimmomatic)
-  for paired end reads:
	- `bash batch-trimmomatic-pe.sh *_1.txt`
	- outputs 4 files: 1\_paired.fq, 2\_paired.fq, 1\_unpaired.fq, 2\_unpaired.fq
-  for single end reads:
	`batch-trimmomatic-se.sh`


### 2) Quality check (FastQC) 
- this step is useful for picking samples to use in your Trinity assembly
- this step is also helpful for PE data, where you use the length of your sequences for merging reads (i.e. FLASH)
-  usage: `bash batch-fastqc.sh *.fastq`


- to move html files from the cluster to your computer
	- open a terminal that is not logged into cluster
	- `rsync user@sherlock.stanford.edu:<path on sherlock>*.html <directory path on your computer>`
	- `rsync -a` will recursively transfer an entire directory

### 3) Merge paired end reads (FLASH) 
- This step is necessary for PE data to merge your two reads (\_1 and \_2) that you will use in transcriptome assembly, but is not necessary for SE data 
- usage: `bash batch-flash.sh *_1_paired.fastq`
- [webpage](https://ccb.jhu.edu/software/FLASH/)
- get lengths of your trimmed seqs above and use in FLASH script


### 4) Pick what samples to assemble into a transcriptome  
- pick samples relevant to your biological question
- restrict sample # for computational capacity (i.e. 3)
	- examples:
		- Balanus natural cline: 3 trinity transcriptomes, one for each population that includes 3 individuals 
		- Coral stress physiology: 1 trinity transcriptome with 2 experimental individuals and 1 control individual


### 5a) De novo assembly (Trinity) 
- You need to hardcode this script for yourself
	- an example: `example_Trinity_mkm.sbatch`
- [Trinity wiki] (https://github.com/trinityrnaseq/trinityrnaseq/wiki)
- to see insides of Trinity: 
	-  	`.Trinity --show_full_usage_info` 
	-  you need to update this example for your assembly 
- 	[Trinity computing requirements](https://github.com/trinityrnaseq/trinityrnaseq/wiki/Trinity-Computing-Requirements)
	-  1GB RAM for 1M reads
	- 	Example: 3 Balanus samples: 11649437 + 11728787 + 11556893
	-  total pairs of reads = 34,835,117 is 35GB RAM and 35 hours of time

### 5b)Check out the quality of your assembly

- to get median contig length:


  ```
  samtools faidx Bg_2assembliesof3.fa
  module load R
  R
  index<-read.delim(‘<file.fa.fai>’, header=F)
  median(index[,2])
  table(cut(index[,2],30)) #bins into groups of 30
  table(index[,2]>300) #how many contigs are greater than 30bp
  ```

- to get basic statistics, including N50
	- `perl abyss-fac.pl <assembly.fa>`

### 6a)Take the longest Isoform from each contig 
- `perl longestisoform_trinity.pl <input.fasta> <output.fasta>`
- this reduces computational load

### 6b)If meta-assembling Trinity assemblies 
- rename contigs in each Trinity file by adding a unique letter to the end of each contig name, for ex: "TRINITYa_####"
- separate Trinity runs will have overlapping names that are not the same contig
- if assembling multiple Trinity assemblies in CAP3 you will have duplicate names
- Then combine all Trinitiy assemblies into one file:
	- `cat Trinityrun1.fa Trinityrun2.fa > all_assemblies.fasta`

### 7)Meta-Assembly (CAP3) 
- [paper](http://genome.cshlp.org/content/9/9/868.full)
- [manual](http://computing.bio.cam.ac.uk/local/doc/cap3.txt)
- CAP3 is an overlap consensus assembler that will merge reads that would not assemble in Trinity due to high heterozygosity
- usage:`sbatch cap3.sh all_assemblies.fasta`

after running:
- merge your contigs and singlets files into one assembly file
	- `cat file.fasta.cap.contigs file.fasta.cap.singlets > newfile.fasta`
	- contigs are what CAP3 merged, singlets did not merge and still contain the Trinity output name
	
- to look at contig #s after CAP3:  
	- `grep -c '>' file.fasta`

- to check for contig name duplicates: 
	- `grep ">" <assembly_file.fa> | perl histogram.pl | head -n`

### 8)Annotate (BLAST) 
-  [Blastx program download](http://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastDocs&DOC_TYPE=Download)
-  [blastx commands, table C4](http://www.ncbi.nlm.nih.gov/books/NBK279675/)

### 8a)Downloading a genome of interest to blast against
- if you don't have this resource; move to 8b
- download cDNA files for blasting your nucleotides against real transcripts
- use the makeblastdb tool from ncbi to make a database:
	- to filter a transcriptome, start by blasting to full genome scaffolds
	- make a nucleotide database:
		- `makeblastdb -in <infile.fasta> -dbtype nucl -out <outfile>`
	- blast to scaffold database with blastn:
	- example scripts: batch-blastn-disco.sh (to split your assembly up for parallel processing), blastn-disco.sh
	- take only the top hit from your blast output:
		`awk '!x[$1]++' file.blast.out > tophits.blast.out`
	- for annotation purposes, you could try blasting to the protein genomic resources:
	- make a protein database:
		`makeblastdb -in <infile> -dbtype prot -out <outfile>`
	- use blastx, example scripts where you will need to change the database to your genome: batch-blastx-uniprot.sh, blastx-uniprot.sh


###Filter Assembly without available genome

### 8b)Annotate with Uniprot database
- 	Uniprot is a more curated database and is recommended over NCBI-nr

#### How to download & create the Uniprot/Trembl database for the first time:
- [download Swiss-Prot & Trembl databases](http://www.uniprot.org/downloads)

	```
	wget <link>
	gunzip <file>.gz
	
	# merge two database files into one database
	cat uniprot_sprot.fasta uniprot_trembl.fasta > unitprot_db.fasta
	
	# make your new fasta into a database
        makeblastdb -in uniprot_db.fasta -dbtype prot -out uniprot_db
   	```

### Run uniprot blast
- usage: `bash batch-blast-uniprot.sh <infile>`
	- the script above splits your assembly into smaller files and calls the blastx on your uniprot database	
- after running, check if you get an error during your blasts
	- `cat slurm*` 
	- most often, your blast may time out
- if you get an error:
 	- `grep -B 1 error slurm*.out`
  	- for any file with error, take line before (the tempfile)
	- `cat <TEMPwerror.fa> <TEMPwerror.fa> > <didnotfinish.fa>`
		- this concatenates all TEMP files that contain an error into a new file
	- you may want to reduce the # of contigs the batch-blast-uniprot.sh script generates for each TEMP file
	- `bash batch-blast-uniprot.sh <didnotfinish.fa>` 

### 8c)Annotate with NCBI-protein database
#### How to download & create the ncbi-protein database for the first time
- Make sure local database on sherlock is up to date
	-  link to ncbi databases: ftp://ftp.ncbi.nlm.nih.gov/blast/db/
	-  nt = nucleotide, nr = protein
	-  for ex: `wget ftp://ftp.ncbi.nlm.nih.gov/blast/db/nr.*.tar.gz`
- to open .tar files downloaded from genbank: 
	- `for i in *.tar ; do tar -xvf $i ; done &`
	- ncbi files are already databases, no need to use makedb script

- To run blast script:
	- `bash batch-blast-ncbiprot.sh input.fasta`
	- this script splits your assembly into TEMP files for parallel processing
- after running, check if you get an error during your blasts
	- `cat slurm*` 
	- your blast search may time out if it runs too long on the cluster
- if you get an error:
 	- `grep -B 1 error slurm*.out`
  	- for any file with error, take line before (the tempfile)
	- `cat <TEMPwerror.fa> <TEMPwerror.fa> > <didnotfinish.fa>`
		- this concatenates all TEMP files that contain an error into a new file
	- you may want to reduce the # of contigs the batch-blast-ncbiprot.sh script generates for each TEMP file
	- `bash batch-blast-ncbiprot.sh <didnotfinish.fa>`


### 9)Parse XML blast files
- Both the uniprot and nr batch scripts above output XML formatted results 
- XML files are difficult to work with, pick what information you want and parse to a tab delimited format
- XML files currently hold the most information about each blast result
	- i.e. gene name, taxonomy 
- to parse uniprot results:
	- `bash batch-parse-uniprot.sh`
	- this script calls parse-uniprot-xml.py on your many temp files
- to parse ncbi-protein results:
 - `bash batch-parse-nr.sh TEMP*.fa.blast.out`

### 10) Use blast results to remove contamination
- If you BLAST to a general database and not to a specific genome, this step is necessary to filter out any contamination
- remove blasts that are likely environmental contamination, i.e. bacteria, fungi, viruses, alveolata, viridiplantae, haptophyceae, etc.

- For uniprot results:
	- first,  merge all of your parsed blast results into one file
		- `cat *_parsed.txt > all_parsed.txt`
	-  second, create a file of only 'good contigs', i.e. metazoans:
		- `bash grep-good-contigs.sh all_parsed.txt assembly.fa`
	- third, if you only want to use uniprot results, pull only good contigs from your assembly fasta file
		- `sbatch batch-filter-assembly.sh assembly.fa goodcontigs.txt`
	- to check how filtering went, count how many contigs you have before and after filtering:
		- `grep -c “>” filteredassembly.fa`

- For ncbi-protein results:
	- first,  merge all of your parsed blast results into one file
		- `cat *_parsed.txt > all_parsed.txt`
	- second, create a taxonomic tree to use for filtering
		- make a tab delimited file with contigs names and their associated GI numbers
			- i.e., import parsed file into excel and keep only those to columns, export as txt file
		- use this GI list to get taxonomic tree 
		- `sbatch batch-taxonID-from-gi.sh type('n' or 'p') GIlist output`
	- third, grep good contigs from uniprot
		- `bash grep-good-contigs-ncbiprot.sh uniprot_goodcontigs.txt`

- combine results from uniprot & ncbi-protein databases:
	- `cat uniprot_goodcontigs ncbi_goodcongits > combined_goodcontigs_duplicates.txt`
	-then take only the uniq contigs from the combined file
	- `cat combined_goodcontigs_duplicates.txt | sort | uniq > combined_goodcontigs_uniq.txt`
	- you can check the # contigs remaining after this to see how filtering went 
	- finally, pull only good contigs from your assembly
	- `sbatch batch-filter-assembly.sh assembly.fa combined_goodcontigs_uniq.txt`

### 10) Assess transcriptome completeness (BUSCO)
- BUSCO provides quantitative measures for the assessment of genome assembly, gene set, and transcriptome completeness, based on evolutionarily-informed expectations of gene content from near-universal single-copy orthologs 

#Install

-  Edit busco's config.ini file to include paths to necessary dependencies on Sherlock

```
	cd ~/.config/
	mkdir busco
	nano config.ini #copy this file from busco's github directory
	#update the following paths in the config.ini file:
	
[tblastn]
# path to tblastn
path = /share/software/user/open/ncbi-blast+/2.7.1/bin

[makeblastdb]
# path to makeblastdb
path = /share/software/user/open/ncbi-blast+/2.7.1/bin
	
[augustus]
# path to augustus
path = /share/software/user/open/augustus/3.3.2/bin/

[etraining]
# path to augustus etraining
path = /share/software/user/open/augustus/3.3.2/bin/

# path to augustus perl scripts, redeclare it for each new script
[gff2gbSmallDNA.pl]
path = /share/software/user/open/augustus/3.3.2/scripts/
[new_species.pl]
path = /share/software/user/open/augustus/3.3.2/scripts/
[optimize_augustus.pl]
path = /share/software/user/open/augustus/3.3.2/scripts/

[hmmsearch]
# path to HMMsearch executable
path = /home/groups/spalumbi/programs/busco-master/hmmer-3.2.1/src/

```	
	

#Run
 - `sbatch busco.sh input.fa output transcriptome`
	
		
## TRANSCRIPTOME ANALYSIS

### Map reads to assembly with Hisat2 (current)

- [Hisat2](https://ccb.jhu.edu/software/hisat2/index.shtml)

```
#create an hisat index
hisat-2 build assembly.fa <output-hisat2-index>
#call the program through a batch script
bash batch-hisat2-fq-paired-eas.sh hisat2-index chunksize *_1.txt.gz

```
- chunksize is how many samples to run for one job, for ex: 4
- for single end reads, batch-hisat2-fq-single.sh



## SNP CALLING & FILTERING

### SNP Calling (BCFtools)
- step 1: `mkdir vcfout`
- usage: `bash bcftools_parallel_eas.sh assembly.fa vcfout cpus *.bam`
- cpus can be something like 24

### Filter SNPs (vcflib)
- [vcflib website](https://github.com/vcflib/vcflib#vcflib)
- [vcflib scripts](https://github.com/vcflib/vcflib/tree/master/scripts)
- can filter for: read depth, read mapping quality, base quality, minor allele frequency, min number of reads mapped, etc.

- First, combine all of your VCF outputs from the parallel processing
`fastVCFcombine.sh combined *.vcf`

- Next, use VCF tools to filter your SNPS:
`sbatch vcftools-snpfilter.sh combined.vcf`
- this script removes indels, takes only biallelic SNPS, requires 7 mapped reads per allele at each SNP, and sets your min/max allele frequencies
- you can also allow some missing data with this script using --max-missing (1 means no missing data allowed, 0 means allow all missing data)


### Create 0,1,2 genotype SNP matrix (vcftools)
- `bash vcftools-012genotype-matrix.sh <combined_filtered_file.vcf> <outfile>`
- this is a simple format used by many downsteams applications

### Protein changes

```
	#get open reading frame predictions for your assembly
	#usage without blastx info:
	sbatch get-orf-predictors.sh assembly.fa empytyfile outfile
	
	#get ORF headers
	grep ">" orf.pep | sed 's/>//g' | sed 's/+//g' > orfpredictor_headers.txt
	
	#get snp protein changes
	snp_protein_changes.py vcf ref.fa orfpredictor_headers.txt OUT
	
```
- this will output a file with 
	- the contig and snp position
	- the reference allele and alternate allele
	- the reference and alternate protein
	- if the protein change was synonymous (S), nonsynonymous (NS), or in a UTR region
	- the position of the change in the codon (CPOS)

### How to subsample a vcf with a list of SNPs names

```
#create a list (goodsnps.txt) of SNP names, for example 'contig/tpos'
awk -v OFS="\t" '{print $1,$2-1,$2}' goodsnps.txt > goodsnps.bed
bedtools intersect -b goodsnps.bed -a allsnps.vcf > goodsnps.vcf
```

###Detect SNP outliers (Bayescan)
- [download program](http://cmpg.unibe.ch/software/BayeScan/download.html)
- make a .bsc file, see R script: `Bayescan_generate_input_file.R`
- rsync .bsc file to sherlock
- on Sherlock cluster, run Bayescan 
	- `sbatch bayescan.sh input.bsc`
- can take SNPs from three separate Bayescan runs
- rsync FST file back to your computer and analyze with R scripts: Bayescan_plot_R.r and Bayescan_plot_results_sample.R
	

## GENE EXPRESSION COUNTS
`bash get-bam-counts.sh *.bam`
- see guide-to-R-scripts.md for more analyses
