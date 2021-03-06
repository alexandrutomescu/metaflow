# MetaFlow - Manual

See a tutorial in [TUTORIAL.md](https://github.com/alexandrutomescu/metaflow/blob/master/TUTORIAL.md).

# 1. Installing and compiling

*MetaFlow* is written in C++ and requires the free [LEMON library](http://lemon.cs.elte.hu). We provide a precompiled version of LEMON in the directory **Src/lemon_binaries_linux**. If you need to use your own compilation of LEMON, update the variable **PATH_TO_LEMON** in *Makefile*.

To install *MetaFlow*, run in the **metaflow** directory:

	make

This will create the executable **metaflow** in the same directory. 


# 2. Preparing the input

*MetaFlow*'s input is a graph-based representation, in LEMON's **LGF** format (LEMON Graph Format), of the alignments of the metagenomic reads in a collection of reference bacterial genomes. We experimented only with *BLAST* alignments (though any aligner could be used). 

## 2.1 Creating a BLAST database and running BLAST
We provide a Python script *Create_Blast_DB.py* which downloads all complete bacterial reference genomes from NCBI ([ftp://ftp.ncbi.nlm.nih.gov/genomes/archive/old_refseq/Bacteria/all.fna.tar.gz](ftp://ftp.ncbi.nlm.nih.gov/genomes/archive/old_refseq/Bacteria/all.fna.tar.gz)), filters out plasmids and transposons, concatenates multiple chromosome of the same species, and keeps only the shortest strain (in case of multiple strains). Run:

	python Create_Blast_DB.py

This creates the files **NCBI_DB/BLAST_DB.fasta** and **NCBI_DB/NCBI_Ref_Genome.txt**. You then have to build a *BLAST* database for **BLAST_DB.fasta**, as follows

	makeblastdb -in NCBI_DB/BLAST_DB.fasta -out NCBI_DB/BLAST_DB.fasta -dbtype nucl

You then align your reads with *BLAST*:

	blastn -query Read_File -out Read_Mappings.blast -outfmt 6 -db NCBI_DB/BLAST_DB.fasta -num_threads 8

## 2.2 Converting BLAST alignments into MetaFlow input

The Python script **BLAST_TO_LGF.py** converts *BLAST*'s output to an input **LGF** file for *MetaFlow*. Run:

	python BLAST_TO_LGF.py Read_Mappings.blast Genome_File Average_Read_Length Sequencing_Machine
	
which produces the file **Read_Mappings.lgf** in the same directory as the **Read_Mappings.blast** file. The parameters are:

- **Read_Mappings.blast**: Blast output file. It must be the tabular format with format=6.
- **Genome_File**: a file containing the genomes in the reference database and their lengths. If you ran the script **Create_Blast_DB.py** above, this was created as **NCBI_DB/NCBI_Ref_Genome.txt**. **If you are using a different or updated database, you need to edit the genome file to incorporate all the reference genomes where the reads have been aligned**. This file format is explained in Section **Genome file** below. 
- **Average_Read_Length**: The average read length of the metagenomics read in the fasta file. 
- **Sequencing_Machine**: Integer value (0 For Illumina, 1 For 454 Pyrosequencing)
	
See Section **Genome File** below if you want to write your own script for converting read alignments (e.g. from a different aligner than *BLAST*) to the **LGF** format required for running *MetaFlow*.
	
#### Example

	python BLAST_TO_LGF.py Example/MCF_Sample_100.blast NCBI_DB/NCBI_Ref_Genome.txt 250 1
	
produces the output file **Example/MCF_Sample_100.blast.lgf**. 

# 3. Running MetaFlow

Run the following command:

	./metaflow -m input.lgf -g Genome_File.txt -c metaflow.config

where

- **input.lgf**: the input file prepared from a read alignment file (see the previous section, **Preparing the input**)
- **Genome_File.txt**: a file containing the genomes in the reference database and their lengths. If you ran the script **Create_Blast_DB.py** above, this was created as **NCBI_DB/NCBI_Ref_Genome.txt**. 
- **metaflow.config**: the configuration file (see below)

#### Example

	./metaflow -m Example/MCF_Sample_100.blast.lgf -g NCBI_DB/NCBI_Ref_Genome.txt -c metaflow.config

## 3.1 Configuration file

You can configure some parameters of *MetaFlow* by editting the file *metaflow.config* (which you must pass to metaflow with the parameter -c). The list of all these parameters and their meaning is described in the Supplementary Material of the paper. The main ones are the following ones. Decrease their value if the sample has low coverage (but false positives may be introduced).

- **REQUIRED_MIN_ABUNDANCE** For any species, if the absolute abundance is lower than REQUIRED_MIN_ABUNDANCE, then that species is considered an outlier and will be removed (default=**0.3**).
- **REQUIRED_AVERAGE_CHUNKS_COVERAGE** For any species, if the average coverage of its chunks (number of reads/number of chunks) is lower than REQUIRED_AVERAGE_CHUNKS_COVERAGE, then the species is considered an outlier and will be removed (default=**2**). 
- **REQUIRED_MAX_PER_OF_EMPTY_CHUNKS** For any species, if the ratio between number of chunks not covered by any read and the total number of chunks is more than REQUIRED_MAX_PER_OF_EMPTY_CHUNKS, then the species will be considered an outlier and will be removed (values in [0..1]) (default=**0.4**). 



# 4. Reading the output

All output files are in CSV format (TAB-separated, not COMMA separated) to make any further analysis easy. They will be generated in the same folder where the input LGF file is located. The main output file is **abundance.csv**. All output files are described below.

|	File 						|	Description																					|
|-------------------------------|-----------------------------------------------------------------------------------------------|
| **abundance.csv** 				| The main output file. It contains the final estimation of the species richness and abundance.	**The abundances are relative to the known species (from the Genome_file)**. |
| **dist.csv** 						| It contains the final distribution of the reads over all chunks in all known genomes.				|
| Step0.abundance.csv			| Intermediary internal files that contain the estimation of the species richness and abundance	|
| Step1.abundance.csv			| Intermediary adundances after Step 1															|
| Step2.abundance.csv			| Intermediary adundances after Step 2															|
| Step3.abundance.csv			| Intermediary adundances after Step 3															|
| Step0.dist.csv				| Intermediary internal files that contains the distribution of the reads over the genome chunks|
| Step1.dist.csv				| Intermediary read distribution after Step 1													|
| Step2.dist.csv				| Intermediary read distribution after Step 2 													|
| Step3.dist.csv				| Intermediary read distribution after Step 3 													|

# 5. Additonal information
## 5.1 Genome file

The genome file contains a list of the bacterial genomes and their length. Do not include plasmids or transposons. **Every genome to where a read was mapped must appear in this file**. Each line has the format:

	GenomeName\tGenomeLength

with the following rules (\t is the TAB character) :

- **GenomeName** is in the format GeneraName_SpeciesName,
- **GenomeLength** is the length of the genome. If one species has different strains with different lengths, select the shortest one.


#### Example

	Campylobacter_jejuni    1718980
	Agrobacterium_radiobacter       6656043
	Burkholderia_ambifaria  14825930
	Leptospira_interrogans  14034030
	Pyrobaculum_oguniense   2436033
	Burkholderia_glumae     6733840


## 5.2 LGF Format

**Read this section if you are writing your own script to convert read alignments into the LGF format needed as input for *MetaFlow* (for example if you are using a different aligner than BLAST).** See also [LGF format](http://lemon.cs.elte.hu/pub/doc/latest/a00002.html) in the LEMON documentation.

The file format is the following one (\t is the TAB character):

#### 5.2.1 Node header

The file starts with two lines marks the beginning of:

	@nodes
	label\tgenome

#### 5.2.2 Genome chunk nodes

Next follows a sequence of lines with the format 

	GenomeId_ChunkNumber\tGenomeId

Consider for exmaple the **Brucella_ovis** genome from the Genome file (see the previous section **Genome file**), and assume that it appear on line **403** in this file. In the LGF file its **GenomeId** is automatically taken its line number - 1, that is, **402**. Its genome length in the file is **1164220**. Its genome is divided into chunks of a speficied size (suppose 2000 for this example); thus it will be divided into **583** chunks (numbered from 0 to 582) for this genome. These 583 chunks should be added as follows:

	402_0	402
	402_1	402
	.....	...
	.....	...
	402_582	402

#### 5.2.3 Read nodes

Next follows a list of all reads in the form 

	ReadName\t-1
	
For example if we have 10 reads named in the fasta file (r1, r2, ..., r10), we add the following lines

	r1	-1
	r2	-1
	..	-1
	..	-1
	r10	-1

#### 5.2.4 Arcs (mappings) header

The next two lines mark the start of mapping from read to chunks.

	@arcs 
	\t\tlabel\tcost

#### 5.2.5 Mappings of reads to genome chunks

If there is a read **r1** that maps to chunk **3** in genome **1**, add a line

	r1\t1_3\tCounter\tCost

Where **Counter** is an integer counter that starts from 0, and **Cost** is a transformation of the read alignment score into a cost (lower is better). Our BLAST_TO_LGF.py script uses the following transformation: cost = |score - max_score| + min_score.

#### 5.2.6 Summary header 

The next line is

@attributes

#### 5.2.7 Summary

If the total number of reads is 100, with average read length 50, and they map to 10 genomes, and maximum cost is 150, minimum cost is 100, add the following lines:

	number_of_genomes	10
	number_of_mapping_reads	100
	avg_read_length	50
	max_cost	150
	min_cost	100


#### Example

	@nodes
	label	genome
	0_0		0
	0_1		0			
	0_2		0
	0_3		0
	0_4		0
	1_0		1
	1_1		1
	1_2		1
	1_3		1
	1_4		1
	2_0		2
	2_1		2
	2_2		2
	r1		-1
	r2		-1
	r3		-1
	r4		-1
	@arcs
			label	cost
	r1	1_0	0	100
	r1	2_1	1	110
	r2	1_1	2	118
	r2	1_0	3	111
	r2	1_0	4	100
	r3	0_2	5	105
	r3	0_3	6	100
	r3	0_4	7	100
	r3	2_2	8	113
	r4	1_3	9	100
	@attributes
	number_of_genomes	3
	number_of_mapping_reads	4
	avg_read_length	50
	max_cost	118
	min_cost	100
	