#Transcriptome assembly & analysis in the Palumbi Lab*


- [Sherlock Wiki] (http://sherlock.stanford.edu/mediawiki/index.php/Main_Page)
- To configure your SSH client to pass those Kerberos credentials for MAC, open terminal (Click Spotlight, type in terminal) and run the following commands:
```
	$mkdir -p ~/.ssh 
	echo "Host sherlock sherlock.stanford.edu sherlock* sherlock*.stanford.edu   
	GSSAPIDelegateCredentials yes   
	GSSAPIAuthentication yes" >> ~/.ssh/config 
```
- create project directory in `$PI_SCRATCH/` (1TB space available)
	- more space and accessible to other users in the lab 
- your personal `$HOME` is 15GB
- [more info on Sherlock storage](http://sherlock.stanford.edu/mediawiki/index.php/DataStorage)

- `kinit user@stanford.edu` & type in pw
- `ssh user@sherlock.stanford.edu` to access cluster
- `$HOME` your personal directory
	 - not accessible to other users
	 - add a program to Sherlock
	 - `cd $PI_HOME/programs`
	 - `wget <linktofiledownload>
	 - `unzip <file>` or `gzip <file.gz>` 
	-  this allows you to access these directories from any directory without having to hardcode the path
	 ```
	 $ cd ~
	 $ nano .bashrc
	 #paste the following into your .bashrc
	 $ export PATH="$PI_HOME/programs:$PATH"
	 $ export PATH="$PI_HOME/scripts:$PATH"
	 $ export PATH="/share/PI/spalumbi/programs/anaconda/bin:$PATH"
	 ```
- 	`login_node` for moving around sherlock
	- one hour time limit
	- use `sdev -t 2:00:00` to get max 2 hrs
-  to run your job on different nodes, in your batch script, add: 
	-  	`#SBATCH -p owners` access to 600 nodes only available to owners. You will be kicked off an owner node if that owner logs on. Always check your slurm file to see if your job was aborted
	-  `#SBATCH -p spalumbi` 256GB memory node
	-  `#SBATCH -p hns` 1TB memory node
- to check on status of job
	- squeue -u username
	- squeue | grep 'spalumbi'
- to cancel a job
	- `scancel <jobID>` to cancel 1 job
	- `scancel -u <username>` will cancel all jobs
- to see how much memory your job has used so far
	- ` sstat --format JobID,NTasks,nodelist,MaxRSS,MaxVMSize,AveRSS,AveVMSize $JOBNUMBER`
 

🌊 `chmod 444` files you don’t want to accidentally write over, 
	`chmod -R ###` for directories

🌊 if downloading a program for the first time, move the executable of the program to the programs folders
 `mv <program> $PI_HOME/programs`

🌊 if downloading a new version of a program, rename with version to not override any executables in that directory

🌊 try to create scripts that are not hard-coded 

🌊 comment your scripts!

	- `cat slurm*`
- when making scripts that split up data into TEMP files for parallel processing, add a line that states your input file in the slurm output
	- `echo $1` if $1 is your input file
- to count contigs in a file
	- `grep -c “>” <filename>`
- this program removes adapters, low quality seqs, etc.
-  for paired end reads:
	- outputs 4 files: 1\_paired.fq, 2\_paired.fq, 1\_unpaired.fq, 2\_unpaired.fq
-  for single end reads:
	`batch-trimmomatic-se.sh`
- this step is useful for picking samples to use in your Trinity assembly, for example: if you have >10 samples you may want to pick those with the highest quality sequencing.
- this step is also helpful for PE data, where you use the length of your sequences for merging reads
	-  	`example_fastqc_MKM.sh` or `fastqc_slurm_barnacles_mkm.sbatch`
- to move a file from cluster to your computer
	`rsync user@sherlock.stanford.edu:/share/PI/spalumbi/... /Users/username/Desktop/...`
	- use `rsync -a` to transfer an entire directory
-  can use the output length stats for your FLASH input
- 	This step is necessary for PE data to merge your two reads (\_1 and \_2) that you will use in transcriptome assembly, but is not necessary for SE data 
-  use your quality outputs from FastQC 
	-  16 Balanus samples from Hopkins
	-  we created 3 separate Trinity assemblies of 3 individuals with best FLASH merging scores
		- the 3 Trinity assemblies ranged from 94,000 - 250,000 contigs
		- after taking the longest isoform of each assembly: 75,000 - 180,000
	-  We then meta-assembled the 3 Trinity assemblies with a long read assembler (CAP3)
		- we found that meta-assembling allowed us to put more samples/diversity into our assembly without inflating the number of contigs
		- for comparison
			- Trinity assembly of 8 individuals: 930,000 contigs and 830,000 longest isoforms
			- CAP3 meta assembly of 3 Trinity assemblies (each made of 3 individuals, 9 individuals total): 200,000 contigs+singlets
	-  	`.Trinity --show_full_usage_info`
	-  you need to update this example for your assembly 
	-  1GB RAM for 1M reads
	-  total pairs of reads = 34,835,117 is 35GB RAM and 35 hours of time

  $ samtools faidx Bg_2assembliesof3.fa
  ```
	- contigs are what CAP3 merged, singlets did not merge and still contain the Trinity output name
	-  `grep -c '>' file.fasta`
	-   `grep ">" <assembly_file.fa> | perl histogram.pl | head -n`
-  [blastx commands, table C4](http://www.ncbi.nlm.nih.gov/books/NBK279675/)

####How to download and create a blast database on your cluster
- download your database
	- ex: uniprot, blast-nr, genome of species of interest
	`for i in *.tar ; do tar -xvf $i ; done &`
- to update an exisitng database (i.e. blast-nr)
	- `perl /share/PI/spalumbi/programs/ncbi-blast-2.3.0+/bin/update_blastdb.pl <database_directory>`

	```
	$ sdev
	$ head <assembly.fa> > <test_assembly.fa>

	```

###8a)Annotate with Uniprot database


#### How to download & create the uniprot database for the first time:

	# merge two files into one database
	$ cat uniprot_sprot.fasta uniprot_trembl.fasta > unitprot_db.fasta
	# make your new fasta into a database
- usage: `bash batch-blast-uniprot.sh infile`
	- the script above splits your assembly into smaller files and calls the blastx on your uniprot database
- 	after running, check if you get an error during your blasts
	- `cat slurm*` 
	- most often, your blast may time out
 	```
 		$ grep -B 1 error slurm*.out
  		# for any file with error, take line before (the tempfile)
		$ this concatenates all TEMP files that contain an error into a new file
		# you may want to reduce the # of contigs the batch-blast-uniprot.sh script generates for each TEMP file
	```		

- Make sure local database on sherlock is up to date
	`for i in *.tar ; do tar -xvf $i ; done &`
	- blast-nr files are already databases, no need to use makedb script
	- `bash batch-blast-nr.sh`
	- splits your assembly into TEMP files for parallel processing
	- `cat slurm*` 
	- most often, your blast may time out
 	```
 	 	$ grep -B 1 error slurm*.out
  		# for any file with error, take line before (the tempfile)
		$ this concatenates all TEMP files that contain an error into a new file
		# you may want to reduce the # of contigs the batch-blast-uniprot.sh script generates for each TEMP file
	```
		
###8c)Reciprocal BLAST 
	-  does heterozygosity cause issues in your alignments

###8d)Downloading a genome of interest to blast against
- ex: Lottia giganteam 
- download cDNA files for blasting your nucleotides against real transcripts
- `makeblastdb `
	- i.e. gene name, taxonomy 
	-  `bash batch-parse-uniprot.sh`
	- `python parse-uniprot-xml.py`
	- `cat *_parsed.txt > all_parsed.txt` #combine all your parsed blast results
-  step 2:
	- create a file of good contigs
	- `sbatch batch-filter-assembly.sh assembly.fa goodcontigs.txt`
	- `grep -c "TRINITY" goodcontigs.txt`

```
	# make a bowtie index from your final assembly
   # check that it outputs 6 files .bt2
   # If you used flash to merge PE reads, use:
   $ bash batch-bowtie2-fq-###.sh b2index 1 *notCombined_1.fastq
   #check TEMPBATCH.sbatch after submitting to see if it started correctly 
   $ cat TEMPBATCH.sbatch
   # if rerunning, make sure you remove files that don't allow writing over, i.e.
   $ rm -metrics.txt 
```

####script options:
- batch-bowtie-fq-flash.sh 
	- if you used flash on your PE reads
- batch-bowtie2-fq-paired.sh

	
##SNP CALLING, FILTERING, & ANALYSIS

```
mkdir vcfout #in same directory as your flash merged samples
sbatch freebayes-cluster.sh assembly.fa vcfout contiglist ncpu *bam

```
-  `$ fastVCFcombine.sh <outfile> *.vcf`
- do this in R
-  easier to do this outside of the cluster
	-  `rsync user@sherlock.stanford.edu:<files> <path to location on your computer>

```
snps<-read.delim('file.012', header=F)
pos<-read.delim('file.012.pos',header=F)
indv<-read.delim<-('file.012.indv',header=F)

colnames(snps)<-paste(pos[,1],pos[,2],sep='-')
rownames(snps)<-indv[,1]
snps<-as.matrix(snps)

#PCA of SNPs
pc.out<-prcomp(snps)
summary(pc.out)
plot(pc.out$x[,1],pc.out$x[,2])	#PC1 v PC2


- script TBD


###Expression counts
- `bash get-bam-counts.sh *.bam`

- in program R
- [website](https://labs.genetics.ucla.edu/horvath/CoexpressionNetwork/Rpackages/WGCNA/)
- script TBD
- use this program in R

library(DESeq2) 
```
- [manual](http://www.htslib.org/doc/samtools.html)
	- `$ samtools tview <sample.bam> <assembly.fa>
	- `$ g` #type in contig name of interest

	- create a file of one contig
	-  `$ samtools view -bh <file.bam> "Contigname" > <outfile.bam>`
-  convert bam to fasta for viewing
		-  `$ samtools bam2fq <infile.bam> | seqtk seq -A > <outfile.fa>`