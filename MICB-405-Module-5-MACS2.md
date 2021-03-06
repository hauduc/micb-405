# Module 5 Worksheet
## ChIP-seq
#### *Axel Hauduc - 09 October 2020*

## Breakout Room Session

### Install IGV for your laptop
http://software.broadinstitute.org/software/igv/download

### Install Bioconda
1. In your home directory, run:

```
mkdir ~/software && cd ~/software
wget https://repo.anaconda.com/archive/Anaconda3-2020.07-Linux-x86_64.sh
./Anaconda3-2020.07-Linux-x86_64.sh
```
Follow all the directions given - choose default parameters. These don't matter and can be changed later.

Then, create your conda environment for MACS2:
```
conda create --name my_macs2_env python=2.7.18
conda activate my_macs2_env
conda install -c bioconda macs2
conda deactivate # when you're done using macs2
```
Every time you will install MACS2, you will need to run ```conda activate my_macs2_env```

Prepare your Orca home directory for tutorial:

```mkdir ~/ChIP_tutorial && cd ~/ChIP_tutorial```


Check the files needed for the tutorial using ```ls -lh /projects/micb405/data/mouse/chip_tutorial/```. These are the files that will be used for the tutorial. They contain a subset of reads from generated for this project. If you want to check out their quality, use FastQC.


What do the “.1” and “.2” in the file names mean?


The (recommended) tools that you will working with to complete this project include:
   * BWA
   * SAMBAMBA (similar in functionality to samtools but general works faster)
   * MACS2 (through Bioconda)
   * IGV (installed on your computer)


### Aligning using BWA MEM

We will now process and map the reads using bwa mem.  Note the run time command syntax for bwa-mem is different (easier and faster) from bwa-aln introduced in class. The alignment process will take ~20min to complete depending on server load. It is highly recommended to construct your commands within a shell script and generated associated log files by redirecting STDOUT as described in class. To make your subsequent commands easier, remember that you can define paths to commonly used files/resources in your script or shell as variables. For example, you can define the path to where the indexed mm10 genome is and the directory to where the tutorial input files are stored (see below). Best practice is to use the explict (a.k.a. absolute) rather than implicit (a.k.a. relative) path when defining these variables, as that will ensure that your script will behave similarly no matter where it is located on your filesystem.

```
GENOME=/projects/micb405/resources/genomes/mouse/mm10/bwa_index/mm10.fa
DATA=/projects/micb405/data/mouse/chip_tutorial
```
You only need to ```export``` a variable if you want to add it to your current shell and have it persist between sessions. Since a script runs a subshell in a continuous session with its own unique variables, it's not necessary to ```export``` variables within it.

As should now be familiar, once defined you can call these paths in your script/shell by entering the special character $ followed by the variable name as shown below. The following command will run run bwa mem and output the .sam file in your current working directory.  You can use this or design your own.

```
bwa mem -t 8 $GENOME ${DATA}/Naive_H3K27ac_1.fastq $DATA/Naive_H3K27ac_2.fastq > ./Naive_H3K27ac.sam 2> bwa_naive_h3k27ac.log &
bwa mem -t 8 $GENOME ${DATA}/Naive_Input_1.fastq $DATA/Naive_Input_2.fastq > ./Naive_Input.sam 2> bwa_naive_Input.log &
```
Note that the -t 8 is to do multithreaded processing to improve speed. The $GENOME specifies the location of reference genome to use. The $DATA/Naive_H3K27ac_1.fastq $DATA/Naive_H3K27ac_2.fastq  specifies the location of the reads. The >./Naive_H3K27ac.sam  indicates the name of the oputput file.
This step will take some time expect the program to run for about 20 mins or longer depending on the server load

### Check files
At the end, you should have something similar to:
```
user01@orca01:~/ChIP_tutorial$ ls -lh
total 8.7G
-rw-r--r-- 1 mhirst orca_users 4.3G Oct  4 05:33 Naive_H3K27ac.sam
-rw-r--r-- 1 mhirst orca_users 4.5G Oct  4 05:36 Naive_Input.sam
-rwxrwx--- 1 mhirst orca_users  479 Oct  4 05:17 bwa_aln.sh
-rwxrwx--- 1 mhirst orca_users  469 Oct  4 05:10 bwa_aln.sh~
-rw-r--r-- 1 mhirst orca_users  22K Oct  4 05:36 bwa_naive_Input.log
-rw-r--r-- 1 mhirst orca_users  23K Oct  4 05:33 bwa_naive_h3k27ac.log
-rw------- 1 mhirst orca_users   58 Oct  4 05:36 nohup.out
```

### Sort, markdup, and index alignments using SAMBAMBA
The output from BWA MEM is an unsorted SAM file. Downstream tools require a sorted BAM file as input, so an intermediate step is to sort and index the alignment to facilitate future steps. To do that you can use tools such as samtools or SAMBAMBA. The commands to do using SAMBAMBA (a faster version of samtools) are the following, for the first sample:

```
sambamba view -S -f bam -o Naive_H3K27ac.bam Naive_H3K27ac.sam
sambamba sort -t 8 Naive_H3K27ac.bam 
sambamba markdup -t 8 Naive_H3K27ac.sorted.bam Naive_H3K27ac.sorted.mkdup.bam
```


NOTE:  Indexes are automatically generated for you by SAMBAMBA 

Do you know how to do the same for the other sample?

### Clean-up intermediate files:
Now is the time to clean-up any intermediate files that are not needed downstream. This would include the .sam, .bam, .sorted.bam file series keeping only the .sorted.mkdup.bam files needed by MACS2. When you are finished cleaning your files the working directory should be ~2.1G total

### Call peaks using MACS2
Before doing peak calling, it is necessary to have a sample dataset, as well as a control dataset. While MACS2 offers the option to perform both regular and broad peak-calling, in this case, we will only be doing regular peak calling, however the instructions are similar for broad peak-calling just add the additional flag --broad to the command.

We will call peaks on sample Naive_H3K27ac.sorted.mkdup.bam and we will be using the Naive_Input.sorted.mkdup.bam as a control (background). Once you have the sorted, dupmarked bam files for all samples, you can perform the MACS2 peakcalling like this:

```
conda activate my_macs2_env
macs2 callpeak -t Naive_H3K27ac.sorted.mkdup.bam -c Naive_Input.sorted.mkdup.bam -f BAMPE -g mm -n Naive_H3K27ac -B -q 0.01
conda deactivate
```
Note that you need to activate your Conda environment in order to use MACS2 now that it is installed in your conda environment

   * The -t is the treament or IP aligned and markdup bam file
   * The -c is the control or INPUT aligned and markdup bam file
   * The -f indicates the input file type (BAM paired-end or BAMPE in this case)
   * The -g indicates the effective genome size (here precomputed for mm10 and provided as ‘mm’)
   * The -n is the of the output file
   * The -B indicates that the program should create a BedGraph (.bdg) file with the results
   * The -q is the FDR cutoff for which to call peaks


Which output file contains the peak information?

### Check files
At the end, you should have something similar to:
```
user01@orca01:~/ChIP_tutorial$ ls -lh
total 3.2G
-rw-r--r-- 1 mhirst orca_users 958M Oct  4 15:20 Naive_H3K27ac.sorted.mkdup.bam
-rw-r--r-- 1 mhirst orca_users 5.6M Oct  4 15:20 Naive_H3K27ac.sorted.mkdup.bam.bai
-rw-r--r-- 1 mhirst orca_users 815M Oct  4 15:57 Naive_H3K27ac_control_lambda.bdg
-rw-r--r-- 1 mhirst orca_users 1.1M Oct  4 15:57 Naive_H3K27ac_peaks.narrowPeak
-rw-r--r-- 1 mhirst orca_users 1.3M Oct  4 15:57 Naive_H3K27ac_peaks.xls
-rw-r--r-- 1 mhirst orca_users 754K Oct  4 15:57 Naive_H3K27ac_summits.bed
-rw-r--r-- 1 mhirst orca_users 310M Oct  4 15:57 Naive_H3K27ac_treat_pileup.bdg
-rw-r--r-- 1 mhirst orca_users 1.1G Oct  4 15:30 Naive_Input.sorted.mkdup.bam
-rw-r--r-- 1 mhirst orca_users 5.8M Oct  4 15:30 Naive_Input.sorted.mkdup.bam.bai
-rwxrwx--- 1 mhirst orca_users  479 Oct  4 05:17 bwa_aln.sh
-rwxrwx--- 1 mhirst orca_users  469 Oct  4 05:10 bwa_aln.sh~
-rw-r--r-- 1 mhirst orca_users  22K Oct  4 05:36 bwa_naive_Input.log
-rw-r--r-- 1 mhirst orca_users  23K Oct  4 05:33 bwa_naive_h3k27ac.log
-rw------- 1 mhirst orca_users   58 Oct  4 05:36 nohup.out
```

### Final Processing (from the end of Thursday, October 8 lecture)
Convert to bigWig and sort (on the Orca server)
```
wget http://hgdownload.cse.ucsc.edu/goldenPath/mm10/bigZips/mm10.chrom.sizes > mm10.chrom.sizes

bedtools sort -i treatment_bedgraph.bdg > treatment_bedgraph.sorted.bdg
bedtools sort -i input_bedgraph.bdg > input_bedgraph.sorted.bdg

bedGraphToBigWig treatment_bedgraph.sorted.bdg mm10.chrom.sizes treatment_bedgraph.sorted.bdg.bw
bedGraphToBigWig input_bedgraph.sorted.bdg mm10.chrom.sizes input_bedgraph.sorted.bdg.bw
```

### Transfer all relevant output files to your local computer
Using a different terminal window that is not connected to the server (if you are using Mac/Linux) or WinSCP (if you are using Windows), retrieve the bed graph files

```
scp user01@orca1.bcgsc.ca:/home/user01/ChIP_tutorial/*{.bw,.chrom.sizes} /path/to/local/folder
```

Load all the data in IGV & launch IGV on your computer.

If you haven’t installed it yet, please get it here IGV download. Make sure you are loading the Mouse (mm10) reference genome by clicking on the drop-down menu on the top left hand corner.

CONGRATULATIONS! You have now completed your first ChIP-seq analysis.  

Now, find one H3K27ac peak. What is its chromosomal position? Based on your two bed.bw files, how do you know it's a peak?
