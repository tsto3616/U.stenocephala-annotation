# **General bioinformatic pipeline for the whole genome sequencing and annotation for *Uncinaria stenocephala*:**

## **File input types:**

The forward and reverse reads are specified according to the end of the file, with \_1 for forward reads and \_2 for reverse reads. This notation will remain throughout the respository.

## **Read processing and assembling:**

fastq files were screened for quality scores of both the JS6871 and JS6872 fastq files. This was performed using the code:

```         
fastqc /scratch/U_steno_seq/Illumina/JS6871*.fq -o /scratch/U_steno_seq/Illumina

fastqc /scratch/U_steno_seq/Illumina/JS6872*.fq -o /scratch/U_steno_seq/Illumina
```

This generates two files per sample, which are located in this branch under the path JS6871/fastqc and JS6872/fastqc. The two files are specified according to the orientation of the reads, \_1\_ and \_2\_ confirm the forward and reverse reads respectively.

It was decided that trimming for the reads was not needed given the good quality scores.

### **Read based estimates of genome statistics:**

It is possible to estimate the basic genome statistics through a kmer based analysis of the reads, which is carried out below through a combination of the programs jellyfish and GenomeScope2.0. The resultant histogram (.histo extension) is then deposited into the GenomeScope2.0 website where it is used output the genome statistics.

```         
jellyfish count -m 21 -C -s 1000000000 -t 10 /scratch/U_steno_seq/Illumina/JS6871_* -o /scratch/U_steno_seq/Illumina/genomescope/reads_JS6871.jf
jellyfish histo -t 10 /scratch/U_steno_seq/Illumina/genomescope/reads_JS6871.jf > /scratch/U_steno_seq/Illumina/genomescope/reads_JS6871.histo
```

The results of these analyses can be viewed in the links below:

JS6871: <http://genomescope.org/genomescope2.0/analysis.php?code=9QCNCJ3TSG9KbMSym9sp>

JS6872: <http://genomescope.org/genomescope2.0/analysis.php?code=3BrQirQDQSqyjwYnKzaP>

### **Reads assembly:**

Assembly of reads was carried out by SPAdes, according to the code below:

```         
spades.py -1 /scratch/U_steno_seq/Illumina/JS6871_1.fq.gz -2 /scratch/U_steno_seq/Illumina/JS6871_2.fq.gz /scratch/U_steno_seq/Illumina/JS6871

spades.py -1 /scratch/U_steno_seq/Illumina/JS6872_1.fq.gz -2 /scratch/U_steno_seq/Illumina/JS6872_2.fq.gz /scratch/U_steno_seq/Illumina/JS6872
```

The contigs and scaffolds of the JS6871 and JS6872 are featured in the JS6871 and JS6872 folders of this branch of the repository. These files are referred to as contigs.fasta and scaffolds.fasta respectively.
