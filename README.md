# Gene prediction and optimization incorporating BIND and MIND workflows:

## Overview of MIND and BIND:

**MIND**: _ab initio_ gene predictions by **M**AKER combined with gene predictions **IN**ferred **D**irectly from alignment of RNA-Seq evidence to the genome.
**BIND**: _ab initio_ gene predictions by **B**RAKER combined with gene predictions **IN**ferred **D**irectly from alignment of RNA-Seq evidence to the genome.

1. Before you use the pipeline: Identify a diverse, orphan-enriched RNA-Seq dataset from NCBI-SRA:
	- Search RNA-Seq datasets for your organism on NCBI, filter Runs (SRR) for Illumina, paired-end, HiSeq 2500 or newer.
	- Download Runs from NCBI (SRA-toolkit, https://www.ncbi.nlm.nih.gov/sra)
	- If gene existing annotations are available: 
	       -expression quantification is done against every gene using every SRR with Kallisto.  ****Arun, I am not sure what this means?
	      -run phylostratR (https://doi.org/10.1093/bioinformatics/btz171) on existing gene models to infer phylostrata of each gene model
	- Rank the SRRs (runs) with highest number of expressed orphans and select feasible amounts of SRR with high numbers of expressed orphans to work with.

	If NCBI-SRA has no RNA-Seq samples of your organism, and you are relying solely on RNA-Seq that you generate yourself, best practice is to maximize representation of all genes by including diverse conditions such as reproductive tissues and stresses, in which orphan gene expression is often high.

2. Run BRAKER
	- Align RNA-Seq with splice aware aligner (STAR or HiSat2 preferred, HiSat2 used here)
	- Generate BAM file for each SRA-SRR id, merge them to generate a single sorted BAM file
	- Run BRAKER


3. Run MAKER
	- Align RNA-Seq with splice aware aligner (STAR or HiSat2 preferred, HiSat2 used here)
	- Generate BAM file for each SRA-SRR id, merge them to generate a single sorted BAM file
	- Run Trinity to generate transcriptome assembly using the BAM file
	- Run TransDecoder on Trinity transcripts to predict ORFs and translate them to protein
	- Download SwissProt curated proteins (use proteins from Viridiplantae for plants)
	- Run MAKER with transcripts (Trinity), proteins (TransDecoder and SwissProt), in homology-only mode
	- Use the MAKER predictions to train SNAP and AUGUSTUS. Self-train GeneMark
	- Run second round of MAKER with the above (SNAP, AUGUSTUS, and GeneMark) ab initio predictions plus the results from previous MAKER rounds.


4. Run Direct Inference evidence-based predictions:
	- Align RNA-Seq with splice aware aligner (STAR or HiSat2 preferred, HiSat2 used here)
	- Generate BAM file for each SRA-SRR id
	- For each BAM file, use multiple transcript assemblers for genome guided transcript assembly:
		* Class2
		* StringTie
		* Cufflinks
	- Run PortCullis to remove invalid splice junctions
	- Consolidate transcripts and generate a non-redundant set of transcripts using Mikado.
	- Predict ORFs on these consolidated transcripts using TransDecoder
	- Pick best transcripts using all the above information with Miakdo Pick.


5. final step in BIND
	- Use Mikado to combine BRAKER-generated predictions with Direct Inference evidence-based predictions.


6. final step in MIND
	- Use Mikado to combine MAKER-generated predictions with Direct Inference evidence-based predictions.




## Steps in detail, with scripts
All case studies used in the manuscript are listed in the **Table** [**here**](case-studies.md). Additional information can be found by clicking on the case study (methods, parameters and dataset used).  

## 1. Finding Orphan Enriched RNAseq dataset form NCBI:

Go to [NCBI SRA](https://www.ncbi.nlm.nih.gov/sra) page and search with "SRA Advanced Search Builder". This allows you to build a query and select the Runs that satisfy certain requirements. For example:

```
("Arabidopsis thaliana"[Organism] AND
  "filetype fastq"[Filter] AND
	"paired"[Layout] AND
	"illumina"[Platform] AND
	"transcriptomic"[Source])
```

Then export the results to "Run Selector" as follows:

![SRA results](Assets/ncbi-sra-1.png)

Clicking the "Accession List" allows you to download all the SRR IDs in a text file format. For downloading data with SRA Toolkit (use [`runSRAdownload.sh`](scripts/runSRAdownload.sh))

![SRA results](Assets/ncbi-sra-2.png)


```bash
while read line; do
	runSRAdownload.sh $line;
done<SRR_Acc_List.txt
# max file size is set to 100Gb
# change `--max-size 100G` to allow >100Gb files
```

Note: depending on how much data you find, this can take a lot of time as well as resources (disk usage). You may need to narrow down and select only a subset of total datasets. One way to do this is by selecting SRRs most likely to be diverse (eg: stress response, flowering tissue, or SRRs with very deep coverage). In this workflow, we will screen them based on number of orphan genes expressed in each SRR. This assumes that you have existing annotations for your organism. If you don't, then you can skip the next few steps and move to "Run BRAKER" section.

Download the CDS sequences for your organism

```
#CDS
wget https://www.arabidopsis.org/download_files/Genes/Araport11_genome_release/Araport11_blastsets/Araport11_genes.201606.cds.fasta.gz
gunzip Araport11_genes.201606.cds.fasta.gz
kallisto index -i ARAPORT11cds Araport11_genes.201606.cds.fasta
```


For each SRR ID, run the Kallisto qualitification ([`runKallisto.sh`](scripts/runKallisto.sh)).

```bash
while read line; do
	runKallisto.sh ARAPORT11cds $line;
done<SRR_Acc_List.txt
```
Once done, the tsv files containing counts and TPM were merged using [`joinr.sh`](scripts/joinr.sh):

```bash
joinr.sh *.tsv >> kallisto_out_tair10.txt
```

For every SRR id, the file contains 3 columns, `effective length`, `estimated counts` and `transcript per million`.

### Run phylostratr to infer phylostrata of genes, and identify orphan genes.

The input to phylostratr is the predicted proteins from your gene-prediction method.
Using this input, run the [`runPhylostrarRa.sh`](scripts/runPhylostratRa.sh). This will download the datasets, but will not run BLAST.
Run Blast using [`runBLASTp.sh`](scripts/runBLASTp.sh) and proceed with formatting the BLAST results using [`format-BLAST-for-phylostratr.sh`](scripts/format-BLAST-for-phylostratr.sh).
Run the [`runPhylostratRb.sh`](scripts/runPhylostratRb.sh).


### Select diverse RNA-Seq data

Once the orphan (species-specific) genes are identified, count the total number of orphan genes expressed (>1TPM) in each SRR, rank them based on % orphan expressed. Depending on how much computational resources you have, you can select the top X number of SRRs to use them as evidence for direct inference and as training data.

Note: for _Arabidopsis thaliana_, we used all of the SRRs that expressed over 60% of the orphan genes (=38 SSRs)
Note: If you are relying solely on RNA-Seq that you generate yourself, best practice is to maximize representation of all genes by including conditions like reproductive tissues and stresses, in which orphan gene expression is high.


## 2A. Run BRAKER (for BIND)

To simplify handling of files, combine all the forward reads to one file and all the reverse reads to another.

```bash
cat *_1.fastq.gz >> forward_reads.fq.gz
cat *_2.fastq.gz >> reverse_reads.fq.gz
```

Run BRAKER using the [`runBRAKER.sh`](scripts/runBraker.sh) script

```bash
runBraker.sh forward_reads.fq.gz reverse_reads.fq.gz TAIR10_chr_all.fas
```

## 2B. Run MAKER (for MIND)

For running MAKER, you will need:


1. Genome assembly in fasta format, eg: `TAIR10_chr_all.fas `
2. Trinity assembled transcripts form the SRA datasets, eg: `trinity.fasta`
3. Trinity transcripts translated to proteins, combined with curated proteins dataset, eg:`trinity-swissprot-pep.fasta`

Here are the steps in detail. generate the CTL files:

```
module load GIF/maker
module rm perl/5.22.1
maker -CTL
```

This will generate 3 CTL files (`maker_opts.ctl`, `maker_bopts.ctl` and `maker_exe.ctl`), you will need to edit them to make changes to the MAKER run.
For the first round, change these lines in `maker_opts.ctl` file:

```
genome=TAIR10_chr_all.fas
est=trinity.fasta
protein=trinity-swissprot-pep.fasta
est2genome=1
protein2genome=1
TMP=/dev/shm
```

Execute MAKER in a slurm file as shown.  It is essential to request more than 1 node with multiple processors to run this efficiently.

```
mpiexec \
	-n 64 \
	/work/GIF/software/programs/maker/2.31.9/bin/maker \
	-base maker \
	-fix_nucleotides
```

Upon completion, train SNAP, AUGUSTUS and Pre-trained GeneMark to run the second round using [script](../scripts/maker_process.sh)

```bash
# process first round results and train SNAP/AUGUSTUS
maker_process.sh maker
# train GeneMark
runGeneMark.sh TAIR10_chr_all.fas
```


Once complete, the second round of MAKER is run by modifying the following lines in `maker_opts.ctl` file:


```
snaphmm=maker.snap.hmm
gmhmm=gmhmm.mod
augustus_species=maker_20171103
```

and run MAKER as:

```bash
mpiexec \
	-n 64
	/work/GIF/software/programs/maker/2.31.9/bin/maker \
	-base maker \
	-fix_nucleotides
```

finalize predictions using [`maker_finalize.sh`](../scripts/maker_finalize.sh) script.

```bash
maker_finalize.sh maker
```

## 3. Evidence (direct inference) gene models.

Cufflinks and Class2 may takes a long time to process BAM file with deep depth region. Instead of using a merged BAM file, single BAM file for each SRR sample is used in multiple transcript assembly programs to generate transcripts . If you plant to use more RNAseq data, you can use the [`runHisat2.sh`](scripts/runHisat2.sh) and generate a single, BAM file of mapped RNAseq reads.

Run Transcriptome assemblies using Class2, Cufflinks and Stringtie. The scripts   [`runClass2.sh`](scripts/runClass2.sh),  [`runCufflinks.sh`](scripts/runCufflinks.sh), [`runStringtie.sh`](scripts/runStringtie.sh) can be executed as:

```bash
while read line; do
	./runClass2.sh ${line}_sorted.bam;
	./runCufflinks.sh ${line}_sorted.bam;
	./runStringtie.sh ${line}_sorted.bam;
done < SRR_Acc_List.txt
```

Class2, Cufflinks and StringTie generates a GFF3 format transcripts for each SRR sample.

The mapped reads can them be processed to generate high confidence splice sites that are useful for picking correct transcritps. We use PortCullis [`runPortCullis.sh`](scripts/runPortCullis.sh) for that purpose:

```bash
runPortCullis.sh merged_SRA.bam
```

We will now consolidate all the transcripts, removing redundancy to create input set of transcripts for Mikado. This is done using Mikado prepare [`runMikado_round1.sh.sh`](scripts/runMikado_round1.sh).

Running this script requires: `list.txt` a file listing all transcript assemblies and weight; `genome.fasta` target genome assembly; `junctions.bed` with splice junctions information.

`list.txt`

```
SRRID1_class.gtf    cs_SRRID1    False
SRRID1_cufflinks.gtf cl_SRRID1     False
SRRID1_stringtie.gtf st_SRRID1    False
SRRID2_class.gtf    cs_SRRID2    False
SRRID2_cufflinks.gtf cl_SRRID2     False
SRRID2_stringtie.gtf st_SRRID2    False
...
```

```bash
runMikado_round1.sh genome.fasta junctions.bed list.txt DI
```

This will generate `DI_prepared.fasta` file that will be used for predicting ORFs using TransDecoder [`runTransDecoder2.sh`](scripts/runTransDecoder2.sh) as follows.

```bash
runTransDecoder2.sh DI_prepared.fasta
# output: DI_prepared.fasta.transdecoder.bed
```

As a final step, we will pick best transcripts for each locus and annotate them as gene using Mikado pick.

```
runMikado-2.sh DI_prepared.fasta.transdecoder.bed DI
```
This will generate:

```
mikado.metrics.tsv
mikado.scores.tsv
DI.loci.gff3
```

## 4A. MIND final step

Merge gene predictions by **M**AKER with gene predictions  **IN**ferred **D**irectly. Here, `maker-final.gff3` generated by MAKER with direct evidence models `DI.loci.gff3`, Mikado can be used in the same way as Step 3, only change the `list.txt`

`list_MIND.txt`
```
maker-final.gff3    mk    False
DI.loci.gff3 DI     False
```

Run Mikado use same scripts as Step 3:

```bash
runMikado_round1.sh genome.fasta junctions.bed list_MIND.txt MIND
runTransDecoder2.sh MIND_prepared.fasta
runMikado-2.sh MIND_prepared.fasta.transdecoder.bed MIND
# Final MIND prediction: MIND.loci.gff3
```

## 4B. BIND final step

Merge gene predictions of **B**RAKER with gene predictions **IN**ferred **D**irectly.
To merge `braker-final.gff3` generated by BRAKER with direct evidence models `DI.loci.gff3`, Mikado can be used in the same way as Step 3, only change the `list.txt`

`list_BIND.txt`
```
braker-final.gff3    br    False
DI.loci.gff3 DI     False
```

Run Mikado use same scripts as Step 3:

```bash
runMikado_round1.sh genome.fasta junctions.bed list_BIND.txt BIND
runTransDecoder2.sh BIND_prepared.fasta
runMikado-2.sh BIND_prepared.fasta.transdecoder.bed BIND
# Final BND prediction: BIND.loci.gff3
```
