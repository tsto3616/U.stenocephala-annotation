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

The scaffolds of the JS6871 and JS6872 are featured in... not sure where to upload these too, will upload to the sharedrive :/B14/Thomas/Uncinaria_assembly/JS6871 /B14/Thomas/Uncinaria_assembly/JS6872.

## **Decontamination:**

Decontamination was carried out through the tool blobtools to remove low coverage and contaminated reads.

First one needs to download the taxonomy dump to provide a reference for blobtools to identify the sequences.

```         
curl -L ftp://ftp.ncbi.nih.gov/blast/db/taxdb.tar.gz | tar xzf -
```

Then to speed up the process the reference file is fixed and split into 100 reads per file.

```         
seqtk seq -A /scratch/U_steno_seq/Illumina/JS6871.fasta/scaffolds.fasta > /scratch/U_steno_seq/Illumina/JS6871.fasta/scaffolds.reduced.l0.fasta
split -l 200 /scratch/U_steno_seq/Illumina/JS6871.fasta/scaffolds.reduced.l0.fasta
```

The split files are then searched via blast, for the database 'nt'.

```         
wget ftp://ftp.ncbi.nlm.nih.gov/blast/db/nt.*.tar.gz

makeblastdb -in /scratch/U_steno_seq/Illumina/nt -dbtype nucl

# Run BLAST in parallel
find . -name 'xaa*' -print0 | xargs -0 -n 1 -P 10 bash -c "blastn -db /scratch/U_steno_seq/Illumina/nt -query \
/scratch/U_steno_seq/Illumina/JS6871.fasta/scaffolds.reduced.l0.fasta -outfmt '6 qseqid staxids bitscore std sscinames \
sskingdoms stitle' -max_target_seqs 10 -max_hsps 1 -evalue 1e-25 -num_threads 16 -out /scratch/U_steno_seq/Illumina/blob.tsv"
```

Coverage data is then generated so that the reads scaffolds can be reduced for low coverage segments.

```         
minimap2 -ax sr -t 16 /scratch/U_steno_seq/Illumina/JS6871/scaffolds.reduced.l0.fasta /scratch/U_steno_seq/Illumina/JS6871_1.fq.gz \
/scratch/U_steno_seq/Illumina/JS6871_2.fq.gz | samtools sort -@16 -O BAM -o /scratch/U_steno_seq/Illumina/Uste_assembly.bam -
```

Blobtools can then be used to eliminate the contaminated data and low coverage data

```         
blobtools create -i /scratch/U_steno_seq/Illumina/JS6871.fasta/scaffolds.reduced.l0.fasta -b /scratch/U_steno_seq/Illumina/Uste_assembly.bam* -t \
/scratch/U_steno_seq/Illumina/blob.tsv -o /scratch/U_steno_seq/Illumina/Uste_norm # 2>&1 | tee /scratch/U_steno_seq/Illumina/blobtools_create.log
blobtools plot -i /scratch/U_steno_seq/Illumina/Uste_norm_out.blobDB.json -r phylum --out /scratch/U_steno_seq/Illumina/export2/Blob_graph2.png \
2>&1 | tee /scratch/U_steno_seq/Illumina/blobtools_plot.png
blobtools view -i /scratch/U_steno_seq/Illumina/Uste_norm_out.blobDB.json -r phylum | gzip > /scratch/U_steno_seq/Illumina/export2/output_view.txt.gz
```

Then we filter the blobtools output to only keep the segments that contain the Nematoda matches or unidentified sequences.

```         
grep "Nematoda\|no-hit" /home/tsto3616/Uste_norm_out.blobDB.table.txt | cut -f1 > /scratch/U_steno_seq/Illumina/blobtools_filter_byhit.list
awk '{if($3>10) print $1}' /scratch/U_steno_seq/Illumina/_Uste_norm_out.Uste_assembly.bam.cov > /scratch/U_steno_seq/Illumina/blobtools_filter_bycov_gt10\
.list
cat /scratch/U_steno_seq/Illumina/blobtools_filter_byhit.list /scratch/U_steno_seq/Illumina/blobtools_filter_bycov_gt10.list | sort | uniq -c | \
awk '{if($1==2) print $2}' > /scratch/U_steno_seq/Illumina/blobtools_filter_bycov_gt10_byhit.list
```

Then we convert the .list file into a fasta file for future analyses:

```         
seqtk subseq /scratch/U_steno_seq/Illumina/JS6871/scaffolds.reduced.l0.fasta /scratch/U_steno_seq/Illumina/blobtools_filter_bycov_gt10_byhit.list > /scratch/U_steno_seq/Illumina/JS6871_cleaned.fasta
```

Then repeat for JS6872!

## **Repeat masking:**

To repeat mask the genome we employed repeat masker. The first step is to build a database for the query/ scaffolds that are going to form the basis of the repeat database used for repeat masking. THen repeat modeler is applied to the database:

```         
BuildDatabase -name /scratch/U_steno_seq/Illumina/JS6872_db /scratch/U_steno_seq/Illumina/JS6872/scaffolds.fasta

RepeatModeler -database /scratch/U_steno_seq/Illumina/JS6872_db -pa 36 > /scratch/U_steno_seq/Illumina/scaffolds-families_2.fa
```

Then to make sense of our database we separate the repeats into known/ classified repeats and unknown repeats. This forms the basis of later repeat annotation if that is to occur.

```         
cat /scratch/U_steno_seq/Illumina/scaffolds-families_2.fa | seqkit fx2tab | awk '{ print "abcDef1_"$0 }' | seqkit tab2fx > \
/scratch/U_steno_seq/Illumina/scaffolds-families.prefix_2.fa

cat /scratch/U_steno_seq/Illumina/scaffolds-families.prefix_2.fa | seqkit fx2tab | grep -v "Unknown" | seqkit tab2fx > \
/scratch/U_steno_seq/Illumina/scaffolds-families.prefix_JS6872.fa.known
cat /scratch/U_steno_seq/Illumina/scaffolds-families.prefix_2.fa | seqkit fx2tab | grep "Unknown" | seqkit tab2fx > \
/scratch/U_steno_seq/Illumina/scaffolds-families.prefix_JS6872.fa.unknown

grep -c ">" /scratch/U_steno_seq/Illumina/scaffolds-families.prefix_JS6872.fa.known
grep -c ">" /scratch/U_steno_seq/Illumina/scaffolds-families.prefix_JS6872.fa.unknown
```

Now that we have our repeat database, we apply this to the file to be masked. To conserve time we decided to use the decontaminated file of JS6872.

```         
RepeatMasker -pa 36 -gff -lib /scratch/U_steno_seq/Illumina/scaffolds-families.fa -dir /scratch/U_steno_seq/Illumina/MaskerOutput /scratch/U_steno_seq/Illumina/JS6871_cleaned.fasta
```

## **Note should include pipeline on rescaffolding if we decide to go down that path due to failure on ONT isolation and amplification!**

## **Searching the genome through BLAST+:**

BLAST+ is a prominent feature of this github repository, with the aim of obtaining results before the assembly and annotation is complete. For this we have outlined the general steps of the search of the local genome, for both blastn and tblastn.

For both the blastn and the tblastn pipeline the original step remains the same, first a database must be built for the genome being searched:

```         
makeblastdb -in /scratch/U_steno_seq/Illumina/JS6872/scaffolds.fasta -dbtype nucl
```

Then we proceed to search the genome for a query of oru choosing, which is listed in the individual branch dedicated for that subset of the analyses. The seach is performed through the code below - first for the blastn search, then the tblastn search:

```         
blastn -task blastn -query /scratch/U_steno_seq/import/PI3K1_nucl.txt -db /scratch/U_steno_seq/Illumina/JS6872/scaffolds.fasta \
-out /scratch/U_steno_seq/Illumina/export2/JS6872_PI3K1_nucl.txt -outfmt "6 qseqid sseqid sseq"

tblastn -task tblastn -query /scratch/U_steno_seq/import/Wnt_path.txt -db /scratch/U_steno_seq/Illumina/JS6872/scaffolds.fasta -out /scratch/U_steno_seq/Illumina/export2/JS6872_PI3K1_prot.txt -outfmt "6 qseqid sseqid sseq"
```

The resultant files are three column text files, with the first column (qseqid) representing the name of the query sequence, the secodn (sseqid) being the name of scaffold containing the query match and the thrid column (sseq) being the match.

We can then input the text file into Rstudio, select out a vector of just the second column, with the code below, then inport the results back into the linux environment to select all the scaffolds in the vector.

```         
# upload matrix
prot<- read.table("C:/Users/tsto3616/U.stenocephala/JS6872_PI3K1_prot.txt")

prot<- prot[, 2]

head(prot)

prot<-as.data.frame(prot)

write.table(prot, file="C:/Users/tsto3616/U.stenocephala/JS6872_PI3K1_names.txt", sep=";", col.names=FALSE, quote=FALSE, row.names=FALSE)
```

Then import it back into Linux to isolate the scaffolds:

```         
seqkit grep -f /scratch/U_steno_seq/import/JS6872_PI3K1_names.txt /scratch/U_steno_seq/Illumina/JS6872/scaffolds.fasta > /scratch/U_steno_seq/Illumina/export2/JS6872_PI3K1_scaff.fasta
```

## **Mapping reads to a reference sequence:**

This practice was used to identify coverage of genes, visualise variation and further confirm the identification of genes. To do this we need to index the reference sequence which is going to have reads mapped to it:

```         
bwa index /scratch/U_steno_seq/Illumina/tubb-1_JS6872.fa
```

Then we map the reads to the reference through bwa mem:

```         
bwa mem /scratch/U_steno_seq/Illumina/tubb-1_JS6872.fa \
/scratch/U_steno_seq/Illumina/JS6872_1.fq.gz /scratch/U_steno_seq/Illumina/JS6872_2.fq.gz > /scratch/U_steno_seq/Illumina/tubb-1_JS6872.sam
```

The .sam file is then reformatted to be visualised

```         
samtools sort /scratch/U_steno_seq/Illumina/tubb-1_JS6872.sam  -o /scratch/U_steno_seq/Illumina/tubb-1_JS6872.sam

samtools view -S -b /scratch/U_steno_seq/Illumina/tubb-1_JS6872.sam > /scratch/U_steno_seq/Illumina/export2/tubb-1_JS6872.bam

samtools sort /scratch/U_steno_seq/Illumina/tubb-1_JS6872.bam -o /scratch/U_steno_seq/Illumina/tubb-1_JS6872.bam

samtools index /scratch/U_steno_seq/Illumina/export/tubb-1_JS6872.bam /scratch/U_steno_seq/Illumina/export/tubb-1_JS6872.bam.bai
```
