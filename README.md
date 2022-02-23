# SemiBin tutorial from scratch

## Install 
```bash
conda create -n semibin_tutorial python=3.8
conda install -n semibin_tutorial -c conda-forge -c bioconda semibin

conda install -c bioconda bowtie2
conda install -c bioconda samtools
conda install -c bioconda megahit
```

## Single-sample binning

### Generating contigs and bam files

Assembly
```bash
megahit -1 sample1_R1.fastq.gz \
-2 sample1_R2.fastq.gz \
--out-dir assembly_contig \
--out-prefix R1
```

Mapping
```bash
bowtie2-build \
-f assembly_contig/R1.contigs.fa assembly_contig/R1.contigs.fa

bowtie2 -q --fr \
-x assembly_contig/R1.contigs.fa \
-1 sample1_R1.fastq.gz \
-2 sample1_R2.fastq.gz \
-S sample1.sam \
-p 64

samtools view -h -b -S sample1.sam -o sample1.bam -@ 64

samtools view -b -F 4 sample1.bam -o sample1.mapped.bam -@ 64

samtools sort \
-m 1000000000 sample1.mapped.bam \
-o sample1.mapped.sorted.bam -@ 64
```

### Easy single binning mode

```bash
SemiBin single_easy_bin \
-i assembly_contig/R1.contigs.fa \
-b sample1.mapped.sorted.bam \
-o easy_single_sample_output 
```

or with pretrained model (for example: sample from human gut)
```bash
SemiBin single_easy_bin \
-i assembly_contig/R1.contigs.fa \
-b sample1.mapped.sorted.bam \
-o easy_single_sample_output_pretrain \
--environment human_gut
```

### Advanced workflows

(1) Generate data.csv/data_split.csv
```bash
SemiBin generate_sequence_features_single \
-i assembly_contig/R1.contigs.fa \
-b sample1.mapped.sorted.bam \
-o advanced_single_sample_output
```

(2) Generate cannot-link
```bash
SemiBin generate_cannot_links \
-i assembly_contig/R1.contigs.fa \
-o advanced_single_sample_output
```

(3) Train
```bash
SemiBin train \
-i assembly_contig/R1.contigs.fa \
--data advanced_single_sample_output/train.csv \
--data-split advanced_single_sample_output/train_split.csv \
-c advanced_single_sample_output/cannot/cannot.txt \
-o advanced_single_sample_output --mode single
```

(4) Bin
```bash
SemiBin bin \
-i assembly_contig/R1.contigs.fa \
--model advanced_single_sample_output/model.h5 \
--data advanced_single_sample_output/data.csv \
-o advanced_single_sample_output
```

with pretrained model
```bash
SemiBin bin \
-i assembly_contig/R1.contigs.fa \
--data advanced_single_sample_output/data.csv \
-o advanced_single_sample_output_pretrain \
--environment human_gut
```


## Co-assembly binning

### Generating contigs and bam files
Assembly
```bash
megahit -1 sample1_R1.fastq.gz,sample2_R1.fastq.gz \
-2 sample1_R2.fastq.gz,sample2_R2.fastq.gz \
--out-dir assembly_contig \
--out-prefix coassembly
```

Mapping
```bash
bowtie2-build \
-f assembly_contig/coassembly.contigs.fa assembly_contig/coassembly.contigs.fa

for sample in {1..2}; \
do \
    bowtie2 -q --fr \
    -x assembly_contig/coassembly.contigs.fa \
    -1 sample"$sample"_R1.fastq.gz \
    -2 sample"$sample"_R2.fastq.gz \
    -S sample"$sample".sam \
    -p 64; \
    
	samtools view -h -b -S sample"$sample".sam -o sample"$sample".bam -@ 64; \
	
	samtools view -b -F 4 sample"$sample".bam -o sample"$sample".mapped.bam -@ 64; \

    samtools sort \
    -m 1000000000 sample"$sample".mapped.bam \
    -o sample"$sample".mapped.sorted.bam -@ 64; \
done
```

### Easy coassembly binning mode

```bash
SemiBin single_easy_bin \
-i assembly_contig/coassembly.contigs.fa \
-b sample*.mapped.sorted.bam \
-o easy_coassembly_sample_output 
```

### Advanced workflows

(1) Generate data.csv/data_split.csv
```bash
SemiBin generate_sequence_features_single \
-i assembly_contig/coassembly.contigs.fa \
-b sample*.mapped.sorted.bam \
-o advanced_coassembly_sample_output
```

(2) Generate cannot-link
```bash
SemiBin generate_cannot_links \
-i assembly_contig/coassembly.contigs.fa \
-o advanced_coassembly_sample_output
```

(3) Train
```bash
SemiBin train \
-i assembly_contig/coassembly.contigs.fa \
--data advanced_coassembly_sample_output/train.csv \
--data-split advanced_coassembly_sample_output/train_split.csv \
-c advanced_coassembly_sample_output/cannot/cannot.txt \
-o advanced_coassembly_sample_output --mode single
```

(4) Bin
```bash
SemiBin bin \
-i assembly_contig/coassembly.contigs.fa \
--model advanced_coassembly_sample_output/model.h5 \
--data advanced_coassembly_sample_output/data.csv \
-o advanced_coassembly_sample_output
```


## Multi-sample binning

### Generating contigs and bam files
Assembly
```bash
for sample in {1..5}; \
do \
    megahit -1 sample"$sample"_R1.fastq.gz \
    -2 sample"$sample"_R2.fastq.gz \
    --out-dir assembly_contig"$sample" \
    --out-prefix R"$sample"; \ 
done
```

Concatenate contigs
```bash
SemiBin concatenate_fasta -i assembly_contig*/R*.fa -o combined_output
```

Mapping
```bash
bowtie2-build \
-f combined_output/concatenated.fa combined_output/concatenated.f5

for sample in {1..5}; \
do \
    bowtie2 -q --fr \
    -x combined_output/concatenated.fa  \
    -1 sample"$sample"_R1.fastq.gz \
    -2 sample"$sample"_R2.fastq.gz \
    -S sample"$sample".sam \
    -p 64; \
    
	samtools view -h -b -S sample"$sample".sam -o sample"$sample".bam -@ 64; \
	
	samtools view -b -F 4 sample"$sample".bam -o sample"$sample".mapped.bam -@ 64; \

    samtools sort \
    -m 1000000000 sample"$sample".mapped.bam \
    -o sample"$sample".mapped.sorted.bam -@ 64; \
done
```

### Easy multi sample binning mode

```bash
SemiBin multi_easy_bin \
-i combined_output/concatenated.fa \
-b sample*.mapped.sorted.bam \
-o easy_multi_sample_output 
```

### Advanced workflows

(1) Generate data.csv/data_split.csv
```bash
SemiBin generate_sequence_features_multi \
-i combined_output/concatenated.fa \
-b sample*.mapped.sorted.bam \
-o advanced_multi_output
```

(2) Generate cannot-link
```bash
for i in {1..5}; \
do \
SemiBin generate_cannot_links \
-i assembly_contig"$i"/R"$i".contigs.fa \
-o advanced_multi_output/samples/R"$i"; \
done
```

(3) Train
```bash
for i in {1..5}; \
do \
SemiBin train \
-i assembly_contig"$i"/R"$i".contigs.fa \
--data advanced_multi_output/samples/R"$i"/train.csv \
--data-split advanced_multi_output/samples/R"$i"/train_split.csv \
-c  advanced_multi_output/samples/R"$i"/cannot/cannot.txt \
-o advanced_multi_output/samples/R"$i" \
--mode single; \
done
```

(4) Bin
```bash
for i in {1..5}; \
do \
SemiBin bin \
-i assembly_contig"$i"/R"$i".contigs.fa \
--model \ advanced_multi_output/samples/R"$sample"/model.h5 \
--data  \ advanced_multi_output/samples/R"$sample"/data.csv \
-o advanced_multi_output/samples/R"$sample"; \
done
```



