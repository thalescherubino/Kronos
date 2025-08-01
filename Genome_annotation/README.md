
# Protein-coding Gene Preidction

## Data availability 
Protein coding genes can be downloaded from Zenodo.  
• `v1.0 annotation`: [https://zenodo.org/records/10215402](https://zenodo.org/records/10215402)  
• `v2.0 annotation`: [https://zenodo.org/records/14189805](https://zenodo.org/records/14189805)  
• `v2.1 annotation [✨Final✨]`: [https://zenodo.org/records/15539216](https://zenodo.org/records/14189805)   

Pretrained ab initio parameters in `Genome_annotation/abinitio_parameters`.  


## Software version
```
trim_galore v0.6.6
cutadapt v3.7
sratoolkit v3.1.1
hisat v2.2.1
stringtie v2.1.7
trinity v2.15.1
pasa v2.5.3
transdecoder v5.7.1
braker v3.0.6
funannotate v1.8.15
segmasker v1.0.0
augustus v3.3.2
snap v2013_11_29
ginger v1.0.1
miniprot v0.12
evidencemodeler v2.1.0
blast v2.15.0
minimap v2.28-r1209
samtools v1.20
isoquant v3.5.2
mikado v2.3.4
busco v5.4.2
omark v0.3.0
```

---

## Annotation v1.0
### 1. Paired-end Short-read Transcriptome Data Processing

This step processes publicly available RNA-seq datasets for Kronos. Reads were downloaded, adapter-trimmed, aligned to the Kronos genome, and assembled both genome-guided and de novo to support gene structure prediction.

**📥 Inputs**  
• `v1_rnaseq.list`: List of NCBI SRA accessions  
• `Kronos.collapsed.chromosomes.fa`: Kronos reference genome v1.0
• `Kronos.collapsed.chromosomes.masked.fa`: Kronos reference genome (masked) v1.0

**📥 Outputs**  
• `all.short-read.rnaseq.merged.sorted.bam`: Merged and sorted RNA-seq alignments by mapping  
• `trinity.transcripts.fasta`: Trinity-assembled transcripts (de novo + genome-guided)   
• `stringtie.gtf`: Genome-guided transcript models from StringTie  
• `sample_mydb_pasa.sqlite.assemblies.fasta`: PASA-refined transcript structures  
• `sample_mydb_pasa.sqlite.assemblies.fasta.transdecoder.genome.gff3`: Translated ORFs from PASA

---

⚙️ **Download RNA-seq datasets from NCBI**  
```
while read -r accession; do 
    sratoolkit.3.1.1-centos_linux64/bin/prefetch ${accession}
    sratoolkit.3.1.1-centos_linux64/bin/fasterq-dump -O . -e ${Numthreads} ${accession}
done < v1_rnaseq.list
```

---

⚙️ **Adapter trimming and quality filtering**  
```
ls *.fastq | cut -d "_" -f 1 | sort -u | while read accession; do 
    trim_galore --paired -j 8 -a "${accession}_1.fastq" "${accession}_2.fastq"
done
```
---
⚙️ **Genome-guided mapping and transcript assembly**  
• Build genome index
```
hisat2-build -p 20 Kronos.collapsed.chromosomes.fa Kronos
```
• Map RNA-seq reads
```
for lib in $(ls *_val_1.fq); do
  prefix=$(echo $lib | cut -d "_" -f 1)
  read_1=$lib
  read_2=$(echo $lib | sed 's/1_val_1/2_val_2/')
  hisat2 -p 56 -x Kronos -1 $read_1 -2 $read_2 --dta -S $prefix.mapped.sam
  echo 'done' > $prefix.done
done
```
• Sort and filter alignments
```
for sam in *.mapped.sam; do
  bam="${sam%.sam}.bam"
  #get primary alignments 
  samtools view -@ 56 -q 20 -h -b -F 260 "$sam" | samtools sort -@ 56 -o "$bam" 
  samtools index "$bam"
done
```
• Merge and assemble with StringTie
```
samtools merge -@ 56 -h SRX10965366.mapped.bam -o all.merged.bam *.mapped.bam #make sure to heve an header
samtools sort -@ 56 all.merged.bam > all.short-read.rnaseq.merged.sorted.bam

stringtie -o stringtie.gtf -p 56 --conservative all.short-read.rnaseq.merged.sorted.bam
```
---
⚙️ **De novo and genome-guided transcriptome assembly with Trinity**  
Note: Due to memory limitations (>1.6 TB input), Trinity was run on individually normalized libraries.  

• Normalize each library
```
#run for for each pair 
singularity run trinity.sif Trinity --verbose --max_memory 90G --just_normalize_reads \
                                    --seqType fq --CPU 40 --left $left --right $right --output trinity_$prefix
```
• Create sample list of normalized reads
```
#list all normalized reads
ls -d trinity_* | while read folder; do
    prefix=$(echo $folder | cut -d "_" -f 2)
    left=$(ls $(pwd)/$folder\/insilico_read_normalization/*_1_val_1*.fq)
    right=$(ls $(pwd)\/$folder\/insilico_read_normalization/*_2_val_2*.fq)
    echo $prefix $prefix $left $right
done > sample.list
```
• Run Trinity (de novo and genome-guided)
```
#merge all outputs into trinity.transcripts.fasta
#run trinity de novo
singularity run trinity.sif Trinity --verbose --seqType fq --max_memory 1500G --CPU 56 --samples_file sample.list

#run trinity genome-guided
singularity run trinity.sif Trinity --verbose --max_memory 250G --CPU 56 --genome_guided_max_intron 10000 --genome_guided_bam all.merged.sorted.bam
```
---
⚙️ **Transcript refinement with PASA**  
```
singularity exec pasapipeline.v2.5.3.simg /usr/local/src/PASApipeline/Launch_PASA_pipeline.pl \
            -c /usr/local/src/PASApipeline/sample_data/sqlite.confs/alignAssembly.config -r -C -R \
            --CPU 56 --ALIGNERS gmap --TRANSDECODER -g Kronos.collapsed.chromosomes.fa \
            -t trinity.transcripts.fasta \ 
            --trans_gtf stringtie.gtf
```
---
### 2. Gene Prediction with BRAKER  
BRAKER was used to generate gene models using both RNA-seq alignment evidence and protein homology. Protein sequences from the Poales clade (TAXID: 38820) were downloaded from UniProt.  

**📥 Inputs**  
• `all.short-read.rnaseq.merged.sorted.bam`: Filtered transcriptome alignments (HISAT2 + SAMtools)  
• `uniprotkb_38820.fasta`: 2.85 million Poales proteins  
• `Kronos.collapsed.chromosomes.masked.fa`: Kronos reference genome (masked) v1.0

**📥 Outputs**  
• `braker.gtf`: Gene models predicted by BRAKER  
• `braker.aa`: protein sequences predicted by BRAKER  

---

⚙️ **Run BRAKER**  
```
singularity exec -B $PWD braker3.sif braker.pl --verbosity=3 \
    --genome=Kronos.collapsed.chromosomes.masked.fa \
    --bam=all.short-read.rnaseq.merged.sorted.bam \
    --prot_seq=uniprotkb_38820.fasta \
    --species=Kronos --threads 48 --gff3 \
    --workingdir=$wd/braker \
    --AUGUSTUS_CONFIG_PATH=$wd/config
```
---

### 3. Gene Prediction with Funannotate  
Funannotate integrates transcriptome evidence. Although it automatically trains ab initio prediction tools, we provided manually trained SNAP and AUGUSTUS models for better accuracy. Training sets were derived from BRAKER models filtered against known references.

**📥 Inputs**  
• `trinity.transcripts.fasta`: Trinity (de novo + genome-guided) assemblies  
• `stringtie.gtf`: StringTie transcript models  
• `all.short-read.rnaseq.merged.sorted.bam`: Filtered transcriptome alignments (HISAT2 + SAMtools)    
• `braker.gtf braker.aa`: BRAKER gene models and proteins  
• `Kronos.collapsed.chromosomes.masked.fa`: Kronos reference genome (masked) v1.0  

**📥 Outputs**  
• `Triticum_kronos.filtered.gff3`: Funannotate gene models


---
⚙️**Manual Training**  
Gene models from BRAKER were filtered to retain only: genes with start & stop codons, full-length hits to the IWGSC reference annotation or translated Trinity transcripts, ≥99.5% sequence identity, and protein length ≥ 350 aa. 6,000 genes were randomly selected for training. See **4. Gene Prediction with Funannotate** for how to train. 
```
#search against trinity or chinese spring protein annotation set
blastp -query braker.aa -db trinity -max_target_seqs -max_hsps 1 -num_threads 56 -evalue 1e-10 -outfmt "6 std qlen slen" -out braker_vs_trinity.blast.out
blastp -query braker.aa -db iwgsc -max_target_seqs -max_hsps 1 -num_threads 56 -evalue 1e-10 -outfmt "6 std qlen slen" -out braker_vs_iwgsc.blast.out
cat  braker_vs_trinity.blast.out braker_vs_iwgsc.blast.out > braker.blast.out

#select genes
python select_genes_for_training.py 6000 
```
---
⚙️ **Run Funannotate**  
```
funannotate predict \
-i Kronos.collapsed.chromosomes.masked.fa \
-o Funannotate \
-s "Triticum kronos" \
--transcript_evidence trinity.transcripts.fasta \
--repeats2evm \
--cpus 56 \
--ploidy 2 \ #2 was used as Kronos is homozygous and can be collapsed 
--rna_bam all.short-read.rnaseq.merged.sorted.bam \
--stringtie stringtie.gtf \
--augustus_species Kronos_manual \
--AUGUSTUS_CONFIG_PATH /global/scratch/users/skyungyong/Software/anaconda3/envs/funannotate/config/ \
--organism other \
--EVM_HOME /global/scratch/users/skyungyong/Software/anaconda3/envs/funannotate/opt/evidencemodeler-1.1.1/ \
--GENEMARK_PATH /global/scratch/users/skyungyong/Software/gmes_linux_64_4 \
```
---
⚙️ **Filter annotations**  
Funannotate produced ~137,000 gene models. Low-complexity ones were filtered post hoc to reduce nosies. 
```
segmasker -in Triticum_kronos.proteins.fa -out Triticum_kronos.proteins.segmakser.out
python filter_genes_funannotate.py
```
---
### 4. Gene Prediction with GINGER  
GINGER uses Nextflow to integrate multiple gene prediction modules. We modified two steps. `denovo.nf`: transcript assembles with oases/velvet were not performed, as this required 17,000,000 Gb memory. `abinitio.nf`: Augustus and SNAP were trained manually.

**📥 Inputs**  
• `trinity.transcripts.fasta`: Trinity (de novo + genome-guided) assemblies  
• `all.short-read.rnaseq.merged.sorted.bam`: Filtered transcriptome alignments (HISAT2 + SAMtools)    
• `braker.gtf braker.aa`: BRAKER gene models and proteins  
• `Protein sequences`: protein sequences downloaded from Ensembl for T. aestivum, T. turgidum, T. dicoccoides, T. spelta, T. urartu 
• `Kronos.collapsed.chromosomes.masked.fa`: Kronos reference genome (masked) v1.0

**📥 Outputs**  
• `ginger_phase2.gff`: GINGER-predicted gene models


---
⚙️ **Manual Training**  
Gene models from BRAKER were filtered to retain only: genes with start & stop codons, full-length hits to IWGSC or translated Trinity transcripts, ≥99.5% sequence identity, and protein length ≥ 350 aa. This time, 8,500 genes were randomly selected for Augustus and SNAP, respectively.   
• Select gene models
```
blastp -query braker.aa -db trinity -max_target_seqs -max_hsps 1 -num_threads 56 -evalue 1e-10 -outfmt "6 std qlen slen" -out braker_vs_trinity.blast.out
blastp -query braker.aa -db iwgsc -max_target_seqs -max_hsps 1 -num_threads 56 -evalue 1e-10 -outfmt "6 std qlen slen" -out braker_vs_iwgsc.blast.out
cat  braker_vs_trinity.blast.out braker_vs_iwgsc.blast.out > braker.blast.out

#select genes
python select_genes_for_training.py 8500 
```

• Train Augustus
```
gff2gbSmallDNA.pl augustus.gff3 Kronos.collapsed.chromosomes.masked.fa 2000 genes.gb
randomSplit.pl genes.gb 400 # 400 test set
new_species.pl --species=Kronos_manual
etraining -species=Kronos_manual genes.gb.train
optimize_augustus.pl --species=Kronos_manual --cpus=48 --UTR=off genes.gb.train
```

• Train SNAP
```
gff3_to_zff.pl genome.dna snap.gff3 > genome.ann # genome.dna = Kronos.collapsed.chromosomes.masked.fa
fathom -validate genome.ann genome.dna 
fathom -categorize 1000 genome.ann genome.dna 
fathom -export 100 -plus uni.ann uni.dna
forge export.ann export.dna
hmm-assembler.pl Kronos . > Kronos_manual.hmm
```

---
⚙️ **Run GINGER**  
```
nextflow -C nextflow.config run mapping.nf
nextflow -C nextflow.config run denovo.nf #with modification
nextflow -C nextflow.config run abinitio.nf #with modification
nextflow -C nextflow.config run homology.nf
phase0.sh nextflow.config
phase1.sh nextflow.config > phase1.log
phase2.sh 50
summary.sh nextflow.config
```

---
### 5. Protein Homology Mapping with Miniprot
Miniprot was used to align 2.85 million Poales protein sequences from UniProt to the Kronos genome to provide homology-based evidence for gene prediction.

**📥 Inputs**  
• `uniprotkb_38820.fasta`: 2,850,097 protein sequences (TAXID: 38820)
• `Kronos.collapsed.chromosomes.masked.fa`: Kronos reference genome (masked) v1.0

**📥 Outputs**  
• `miniprot.gff3`: Protein alignments

---
⚙️ **Run MiniProt**  
```
miniprot -t 56 --gff --outc=0.95 -N 0 Kronos.collapsed.chromosomes.fa uniprotkb_38820.fasta > miniprot.gff3
```

---
### 6. Integration with EvidenceModeler (EVM)
All gene prediction, transcript, and protein evidence was integrated using EvidenceModeler (EVM) to produce consensus gene models.  

**📥 Inputs**  
• `braker.gff`: BRAKER gene models  
• `Triticum_kronos.filtered.gff3`: Funannotate gene models  
• `ginger_phase2.gff`: GINGER gene models  
• `ginger_spaln.gff`: protein alignment models from GINGER  
• `miniprot.gff3`: Protein evidence (Miniprot)  
• `sample_mydb_pasa.sqlite.pasa_assemblies.gff3`: PASA transcript assemblies  
• `sample_mydb_pasa.sqlite.assemblies.fasta.transdecoder.genome.gff3`: Translated ORFs from PASA  
• `repeat.gff3`: repeat annotations from HiTE  
• `Kronos.collapsed.chromosomes.fa`: Kronos reference genome v1.0 

**📥 Outputs**  
• `Kronos.EVM.gff3`: Consensus gene models

---
⚙️ **Input Preprocessing**  
```
# Combine draft predictions
cat braker.gff Triticum_kronos.filtered.gff3 ginger_phase2.gff \
  sample_mydb_pasa.sqlite.assemblies.fasta.transdecoder.genome.gff > abinitio.gff3

# Combine protein alignments
cat miniprot.gff3 ginger_spaln.gff > homology.gff3

# PASA transcript assemblies
cp sample_mydb_pasa.sqlite.pasa_assemblies.gff3 transcripts.gff3
```

---
⚙️ **Run EVM**  
EvidenceModeler was run as below with the specified weights. 
```
singularity exec EVidenceModeler.v2.1.0.simg EVidenceModeler --sample_id Kronos \
            --genome Kronos.collapsed.chromosomes.fa --weights weights.txt --gene_predictions abinitio.gff3 \
            --protein_alignments homology.gff3 --transcript_alignments transcripts.gff3 \
            --repeats repeat.gff3 --CPU 56 -S --segmentSize 100000 --overlapSize 10000
```

• weights.txt
```
ABINITIO_PREDICTION     funannotate   3       #funannotat gene models
ABINITIO_PREDICTION     braker  3             #braker gene models
ABINITIO_PREDICTION     ginger  3             #ginger gene models
ABINITIO_PREDICTION     gingers  3.5          #single-exon genes from GINGER
PROTEIN homology        2                     #ginger's SPALN
PROTEIN                  miniprot       1     #protein mapping with miniprot
OTHER_PREDICTION        transdecoder    2.5   #pasa transcript assembles translated by transdecoder
TRANSCRIPT               pasa  8              #pasa transcript assemblies
```
---
### 7. UTR and Isoform Refinement with PASA
PASA was rerun to update the EVM models with untranslated regions (UTRs) and alternative splicing isoforms.

**📥 Inputs**  
• `Kronos.EVM.gff3`: Consensus gene models  
• `trinity.transcripts.fasta`: Trinity (de novo + genome-guided) assemblies    
• `stringtie.gtf`: StringTie transcript models    
• `Kronos.collapsed.chromosomes.fa`: Kronos reference genome v1.0 

**📥 Outputs**  
• `Kronos.EVM.pasa.gff3`: Final annotation with UTRs and isoforms

---
⚙️ **Run PASA**   
```
#create DB
singularity exec pasapipeline.v2.5.3.simg /usr/local/src/PASApipeline/Launch_PASA_pipeline.pl \
            -C -R -c alignAssembly.config -g Kronos.collapsed.chromosomes.masked.fa \
            -t trinity.transcripts.fasta --trans_gtf stringtie.gtf --TRANSDECODER --ALT_SPLICE --ALIGNERS gmap
#update annotations
singularity exec pasapipeline.v2.5.3.simg /usr/local/src/PASApipeline/Launch_PASA_pipeline.pl \
            -A -L -c compare.config  -g Kronos.collapsed.chromosomes.masked.fa \
            -t trinity.transcripts.fasta --annots Kronos.EVM.gff3
```

---
### 8. Final Gene Model Selection
High-confidence genes were selected by searching final annotations against a panel of known proteins from related grass species.

**📥 Inputs** 
• `Kronos.EVM.pasa.gff3 Kronos.EVM.pasa.pep.fa`: Final PASA-refined annotations  
• `pasa.transdecoder.pep.complete.fa`: Complete ORFs (start + stop codons)  
• `protein evidence datasets`: from Ensembl Plants 
```
Aegilops_tauschii.Aet_v4.0.pep.all.fa Avena_sativa_ot3098.Oat_OT3098_v2.pep.all.fa Avena_sativa_sang.Asativa_sang.v1.1.pep.all.fa Brachypodium_distachyon.Brachypodium_distachyon_v3.0.pep.all.fa  
Triticum_aestivum.IWGSC.pep.all.fa Triticum_urartu.IGDB.pep.all.fa Triticum_dicoccoides.WEWSeq_v.1.0.pep.all.fa Triticum_spelta.PGSBv2.0.pep.all.fa Triticum_turgidum.Svevo.v1.pep.all.fa    
Lolium_perenne.MPB_Lper_Kyuss_1697.pep.all.fa Secale_cereale.Rye_Lo7_2018_v1p1p1.pep.all.fa Hordeum_vulgare.MorexV3_pseudomolecules_assembly.pep.all.fa Hordeum_vulgare_goldenpromise.GPv1.pep.all.fa  
```

---
⚙️ **Pick annotations**  
```
#search against the protein databases
blastp -query Kronos.EVM.pasa.pep.fa -num_threads 56 -evalue 1e-10 -max_target_seqs 10 -max_hsps 1 -outfmt "6 std qlen slen" -out Kronos.v1.0.against.all.all_prot.max10 -db all_prot 
blastp -query Kronos.EVM.pasa.pep.fa -num_threads 56 -evalue 1e-10 -max_target_seqs 10 -max_hsps 1 -outfmt "6 std qlen slen" -out Kronos.v1.0.against.all.pasa_orf.max10 -db pasa_orf

#create version 1 annotations
#the input gff has to be correctly sorted first
python generate_v1.0_annot.py
```

----

## Annotation v2.0

Annotation v2.0 integrates publicly available long-read transcriptome data for *Triticum* to improve annotation accuracy. Assemblies were performed with both StringTie and IsoQuant, incorporating short-read evidence for improved splice junction support.

### 1. Long-read Transcriptome Data Download

**📥 Inputs**  
• `v2_rnaseq.list`: List of NCBI SRA accessions  

**📥 Outputs**  
• `fastq files`: long-read transcriptome data  

---
⚙️ **Data Download**    
```
while read -r accession; do 
    sratoolkit.3.1.1-centos_linux64/bin/prefetch ${accession}
    sratoolkit.3.1.1-centos_linux64/bin/fasterq-dump -O . -e ${Numthreads} ${accession}
done < v2_rnaseq.list
```
---
### 2. Long-Read Alignment with Minimap2  

**📥 Inputs**  
• `fastq files`: long-read transcriptome data  
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: broken Kronos reference genome v1.1 (masked)    

**📥 Outputs**  
• `all.long-read.merged.bam`: long-read transcriptome alignments  

---
⚙️ **Alignment with Minimap2**   
```
for fq in *.fq; do
  prefix=$(echo $fq | cut -d "." -f 1)
  minimap2 -I 12G -t 56 -x splice:hq -a -o "${prefix}.sam" Kronos.collapsed.chromosomes.masked.v1.1.broken.fa ${fq}
  samtools view -@56 -h -b "${prefix}.sam" | samtools sort -@ 56 > "${prefix}.bam"
done
```
• Merge all bam files 
``` 
samtools merge -@56 all.long-read.merged.bam *.bam
samtools index -@56 all.long-read.merged.bam
```
• Split chromosomes for parallel processing   
```
#separate for each chromosome
for chromosome in \
  1A 1A_296625417 1B 1B_362283996 2A 2A_389606086 2B 2B_416081101 \
  3A 3A_354343362 3B 3B_427883679 4A 4A_376933649 4B 4B_351648618 \
  5A 5A_305547233 5B 5B_360298581 6A 6A_294206980 6B 6B_365632995 \
  7A 7A_370147894 7B 7B_378890030 Un; do
  samtools view -@56 -h -b all.long-read.merged.bam $chromosome > all.long-read.merged.${chromosome}.bam
done
```

### 3. Transcript Assembly with StringTie & IsoQuant  

**📥 Inputs**  
• `all.short-read.rnaseq.merged.sorted.*.bam`: Filtered short-read transcriptome alignments produced during v1.0 anotation  
• `all.long-read.merged.*.bam`: long-read transcriptome alignments  
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: broken Kronos reference genome v1.1 (masked)    

**📥 Outputs**  
• `stringtie.denovo.transcript_models.gtf`: transcripts from StringTie
• `isoquant.OUT.transcript_models.gtf`: transcripts from Isoquant

---
⚙️ **Assembly with StringTie**  
```
#make sure to adjust the coordinates for the original genome
stringtie -p 4 -v -o Kronos.${chromosome}.stringtie.denovo.gtf \
         --mix all.short-read.rnaseq.merged.sorted.${chromosome}.bam all.long-read.merged.${chromosome}.bam
```

⚙️ **Assembly with Isoquant**  
```
#make sure to adjust the coordinates for the original genome
isoquant.py --threads 56 --reference Kronos.collapsed.chromosomes.masked.v1.1.broken.fa \
            --illumina_bam all.short-read.rnaseq.merged.sorted.bam --output Isoquant_Kronos --data_type pacbio_ccs --bam $bam #a list of unmerged bamfiles was given
```

### 4. Combining Annotations
**📥 Inputs**  
• `stringtie.denovo.transcript_models.gtf`: transcripts from StringTie  
• `isoquant.OUT.transcript_models.gtf`: transcripts from Isoquant  
• `all.short-read.rnaseq.merged.sorted.bam`: Filtered short-read transcriptome alignments produced during v1.0 anotation    
• `all.evidnece.fa`: protein evidence datasets from Ensembl Plants used during v1.0 anntotation (step 8)  
• `Kronos.collapsed.chromosomes.masked.v1.1.fa`: broken Kronos reference genome v1.1 (masked)      
 

**📥 Outputs** 
• `mikado.loci.coding.gff3`: protein-coding genes from mikado  

---
⚙️ **Set Up Mikado**
```
mikado configure --list list.txt --reference Kronos.collapsed.chromosomes.masked.v1.1.fa \
        --mode STRINGENT --scoring plants.yaml  --copy-scoring plants.yaml –junctions junctions.bed \
        -bt all.evidence.clean.fa configuration.yaml
mikado prepare --json-conf configuration.yaml
```
⚙️ **Create Intermediate Inputs**
```
#get ORFs
prodigal -i mikado_prepared.fasta -g 1 -o mikado.orfs.gff3 -f gff

#get homology
diamond blastx --max-target-seqs 5 --outfmt 5 --more-sensitive --masking 0 \
        --threads 56 --db all.evidence.fa.dmnd --query mikado_prepared.fasta --out mikado_prepared.fasta.xml

#get junction
portcullis full -o all_merged.repositioned.bed -t 20 Kronos.collapsed.chromosomes.masked.v1.1.broken.fa all.short-read.rnaseq.merged.sorted.bam
```
⚙️ **Finish Mikado**
```
mikado serialise --genome Kronos.collapsed.chromosomes.masked.v1.1.fa -p 56 --junctions all_merged.repositioned.bed --orfs mikado.orfs.gff3 -bt all.evidence.clean.fa
mikado pick --configuration configuration.yaml -p 56 --scoring-file plant.yaml --subloci-out mikado.subloci.gff3 --loci-out mikado.loci.out
```
---

### 5. Finalizing Annotation    
We manually and visually compared gene models from Mikado to those in v1.0 annotations (including the final set, BRAKER, Funannotate, and GINGER), as well as long-read-derived transcripts from StringTie and IsoQuant. This step focused on resolving issues such as overlapping exons (CDS and UTR), unusually long introns (both within and outside coding regions), single-exon genes, paralog clusters, and potential chimeric structures.

We looked for annotation-specific patterns and iteratively replaced, removed, or updated subsets of Mikado models to improve structural accuracy. This process involved over 20 rounds of visual inspection and refinement, making it difficult to document as an automated workflow.

### 6. Changes from v1.0 to v2.0  
While most genes retained consistent identifiers across versions (e.g., TrturKRN1A01G000005 → TrturKRN1A02G000005), a small subset of gene IDs were reassigned to accommodate newly defined gene models and structural updates.

If you're working across both v1.0 and v2.0 annotations, we strongly recommend confirming gene homology and genomic coordinates to ensure accurate correspondence between versions.

---
## Annotation v2.1  
In this version, existing genes in the v2.0 annotation that overlap with low-confidence NLRs were discarded. High and medium-confidence NLRs were added instead.   

**📥 Inputs**   
• `Kronos.v2.0.gff3`: Kronos annotation v2.0  
• `Kronos_all.NLRs.final.gff3`: Kronos NLR annotations(https://zenodo.org/records/15539721)  

**📥 Outputs**   
• `Kronos.v2.1.gff3`: Kronos annotation v2.1  

---
⚙️ **Generate annotation**  
• Get NLR confidence  
```
awk -F'\t' '{
  split($9, info, ";")
  id = ""; conf = ""
  for (i in info) {
    if (info[i] ~ /^ID=/) {
      sub(/^ID=/, "", info[i])
      id = info[i]
    }
    if (info[i] ~ /^Confidenice=/) {
      sub(/^Confidenice=/, "", info[i])
      conf = info[i]
    }
  }
  if (id != "" && conf != "") print id "\t" conf
}' Kronos_all.NLRs.final.gff3 > Kronos_all.NLRs.final.conf.list
```
• create annotation v2.1  
```
python generate_v2.1_annot.py
```
• Sort gff  
```
grep -v '^#' Kronos.v2.1.initial.gff3 | \
awk 'BEGIN {OFS="\t"} { 
    if ($3 == "gene") type = 1;
    else if ($3 == "mRNA") type = 2;
    else if ($3 == "exon") type = 3;
    else if ($3 == "CDS") type = 4;
    else type = 5;
    print $0, type 
}' | sort -k1,1 -k4,4n -k10,10n | cut -f1-9 > Kronos.v2.1.gff3  # Sort by chromosome, start, then type order
```


---
## Annotation Assessment
Genome annotations were evaluated by BUSCO and OMArk  

**📥 Inputs**   
• `Kronos.vX.X.pep.fa`: Kronos annotation

⚙️ **BUSCO**  
```
busco -m prot -c 56 -l poales_odb10 -i Kronos.vX.X.pep.fa -o busco
```

⚙️ **OMArk**  
```
#for longest protein per gene
omamer search --db  LUCA.h5 --query Kronos.vX.X.pep.fa --out Kronos.vX.X.pep.fa.omamer
omark -f Kronos.vX.X.pep.fa.omamer -d LUCA.h5 -oomark_output
```
