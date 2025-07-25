# Capture sequencing data remapping 

## Data availability 
Processed exome and promoter-capture data are available through Zenodo   
• `Exome capture (MAPS) [✨Final✨]`: https://zenodo.org/records/15801888  
• `Exome capture (GATK) [✨Final✨]`: https://zenodo.org/records/15801566  
• `Promoter capture (MAPS) [✨Final✨]`: https://zenodo.org/records/15801888    
• `Promoter capture (GATK) [✨Final✨]`: https://zenodo.org/records/15801566  

## Software version
```
sra-toolkit v3.1.1
fastp v0.23.4
trimmomatic v0.39
bwa v0.7.18-r1243-dirty
samtools v1.20
picard v3.0.0
MAPS
GATK v4.5.0
snpeff v5.2a
```
---

## Exome-capture Data Remapping

Exome capture sequencing datasets were  generated by [Krasileva et al., 2017](https://www.pnas.org/doi/10.1073/pnas.1619268114) and were processed using the MAPS pipeline and GATK, separately. 

### 1. Downloading Sequencing Data

📥 Inputs  
• `exome_MAPS_groups.list`: SRA identifiers for exome-capture data 
Note: this list does not contain redundant accessions. Download everything from PRJNA258539, except for SRX2433935 (Kronos3690; SRR5118653) from which we found essentially no mutations.  

📥 Outputs  
• `*.fastq`: paired-end fastq files  

```
#the SRR accessions in **exome_MAPS_groups.list** is now referred to as ${accession}. 
sratoolkit.3.1.1-centos_linux64/bin/prefetch ${accession}
sratoolkit.3.1.1-centos_linux64/bin/fasterq-dump -O . -e ${Numthreads} ${accession}
```
---

### 2. Quality Control and Filtering

📥 Inputs  
• `*.fastq`: paired-end fastq files  

📥 Outputs  
• `*.filtered.fastq`: filtered and trimmed paired-end fastq files  

```
fastp --in1 ${accession}_1.fastq --in2 ${accession}_2.fastq --out1 ${accession}.1.filtered.fq --out2 ${accession}.2.filtered.fq --thread 16 -q 20
```

---
### 3. (MAPS) Read Alignment

📥 Inputs   
• `*.filtered.fastq`: filtered and trimmed paired-end fastq files   
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: Kronos reference genome v1.1 
Note: to learn more about how the reference genome was broken, see [here](https://github.com/krasileva-group/Kronos_EDR)).  

📥 Outputs   
• `*.sam`: alignment files  

```
#index
bwa index Kronos Kronos.collapsed.chromosomes.masked.v1.1.broken.fa
```

```
#for each accession
for i in 1 2; do
    bwa aln -t ${Numthreads} ${reference_dir}/Kronos -f ${accession}.${x}.sai ${accession}.${x}.filtered.fq
done
bwa sampe -N 10 -n 10 -f ${accession}.sam ${reference_dir}/Kronos ${accession}.1.sai ${accession}.2.sai ${accession}.1.filtered.fq ${accession}.2.filtered.fq
```
---
### 4. (MAPS) Sorting and Deduplication

📥 Inputs   
• `*.sam`: alignment files  

📥 Outputs    
• `*.sorted.rmdup.bam`: sorted alignment files without duplicate reads  

```
#sort and index
samtools view -@ ${Numthreads} -h -b ${accession}.sam | samtools sort -@ ${Numthreads} > ${accession}.sorted.bam
samtools index ${accession}.sorted.bam
```

```
#occasionally, the bam file may contains records that picard does not like to process.
#ALIDATION_STRINGENCY=LENIENT can be added to skip those alignments
picard MarkDuplicates REMOVE_DUPLICATES=true I=${accession}.sorted.bam O=${accession}.sorted.rmdup.bam M=${accession}.rmdup.txt
```
---
### 5. (MAPS) Merging Redundant Libraries

📥 Inputs   
• `*.sorted.rmdup.bam`: sorted alignment files without duplicate reads  
• `exome_merge.list`: a list of alignments to be merged together 

📥 Outputs    
• `*.sorted.rmdup.bam`: merged sorted alignment files without duplicate read  

```
#Some mutants have multiple sequencing datasets. These are merged into a single BAM file prior to the MAPS piepline
while read mutant lib1 lib2; do
     samtools merge -@ 56 -o ../sorted.rmdup.bam_files/${lib1}.sorted.rmdup.bam ${lib1}.sorted.rmdup.bam ${lib2}.sorted.rmdup.bam
 done < exome_merge.list
```
---
### 6. (MAPS) Preparing for MAPS Pipeline

📥 Inputs   
• `exome_MAPS_groups.list`: A list of alignments to be processed together as a batch 

📥 Outputs    
• `MAPS-{x}`: folders containing each batch of alignments to be processed together

```
#create folders and move bam files
for i in {1..60}; do mkdir MAPS-${i}; done

# Move BAM files into their respective groups
cat exome_MAPS_groups.list | while read line; do
    accession=$(echo $line | awk '{print $4}')
    i=$(echo $line | awk '{print $6}')
    mv ${accession}.sorted.rmdup.bam MAPS-${i}/
done
```

---
### 7. (MAPS) Running the MAPS Pipeline

📥 Inputs   
• `*.sorted.rmdup.bam`: sorted alignment files without duplicate reads  

📥 Outputs    
• `Kronos_mpileup.txt`: mpile up output for the MAPS pipeline part 1 

⚙️ **Mpileup**  
Note: the MAPS pipeline is available [here](https://github.com/DubcovskyLab/wheat_tilling_pub). Use Python v2.7
```
#for each folder
python ./wheat_tilling_pub/maps_pipeline/beta-run-mpileup.py \
      -t 30 -r Kronos.collapsed.chromosomes.masked.v1.1.broken.fa \
      -q 20 -Q 20 -o Kronos_mpileup.txt -s $(which samtools) \
      --bamname .sorted.rmdup.bam
```

⚙️ **MAPS Part 1**  
Note: set l as **(# libraries - 4)**.
```
mkdir MAPS && cd MAPS
for idx in {2..30}; do # 28 chromosomes + unplaced 
    bed="../temp-region-${idx}.bed"
    mpileup="../temp-mpileup-part-${idx}.txt"

    # Read the first line only and get the first field (chromosome)
    chr=$(awk 'NR==1 {print $1}' "$bed")

    # Check if the chromosome directory exists, if not, create it
    [ ! -d "${chr}" ] && mkdir -p "${chr}"

    # Copy the mpileup file into the appropriate chromosome directory
    # Add error checking for the copy operation
    head -n 1 ../Kronos_mpileup.txt > "${chr}/${chr}_mpileup.txt"
    cat "$mpileup" >> "${chr}/${chr}_mpileup.txt"
done
```

```
for chr in $(ls -d */ | sed 's/\/$//'); do
    pushd "$chr"  
    python ./wheat_tilling_pub/maps_pipeline/beta-mpileup-parser.py -t 56 -f "${chr}_mpileup.txt"
    python ./wheat_tilling_pub/maps_pipeline/beta-maps1-v2.py -f "parsed_${chr}_mpileup.txt" -t 56 -l ${l} -o "${chr}.mapspart1.txt"
    popd
done
```

⚙️ **MAPS Part 2**  
```
cd .. && mkdir MAPS_output && cd MAPS_output
head -n 1 ../MAPS/1A/1A.mapspart1.txt > all.mapspart1.out
#merge all outcomes
for file in ../MAPS/*/*.mapspart1.txt; do tail -n +2 "$file"; done >> all.mapspart1.out

#diversify two parameters
#each pair is homMinCov,hetMinCov
for pair in "2,3" "3,4" "3,5" "4,6"; do
  k=$(echo $pair | cut -d',' -f1)
  j=$(echo $pair | cut -d',' -f2)
  python ./wheat_tilling_pub/maps_pipeline/maps-part2-v2.py -f all.mapspat1.txt --hetMinPer 15 -l $l --homMinCov $k --hetMinCov $j -o all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.tsv -m m
done
```
---
### 8. (MAPS) VCF Conversion

📥 Inputs   
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.tsv`: called mutations in tsv format from four pairs of parameters

📥 Outputs    
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.reformatted.tsv`: reformatted mutations


```
#concatnate all outputs
for tsv in MAPS-1/MAPS_output/*.tsv; do
  cat MAPS-*/MAPS_output/${tsv} > ${tsv}
done

#adjust coordinates for mutation: broken genome -> full genome
#change identifier: SRRXXX -> Kronos3412
ls *.tsv | while read line; do python reformat_maps2_tsv.py $line exome_MAPS_groups.list; done
```
---
### 9. (MAPS) Separate Regions with Residual Hetrogenity

📥 Inputs   
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.reformatted.tsv`: reformatted mutations

📥 Outputs    
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.reformatted.corrected.10kb_bins.RH.byContig.MI.No_RH.maps.vcf`: mutations not from residual hetrogenity regions  
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.reformatted.corrected.10kb_bins.RH.byContig.MI.RH_only.maps.vcf`: mutations from residual hetrogenity regions  


```
Separate regions with residual hetrogenity.
ls *.reformatted.tsv | while read line; do bash ./wheat_tilling_pub/postprocessing/residual_heterogeneity/generate_RH.sh $line chr.length.list; done

mkdir No_RH
mv *No_RH.maps* No_RH/ && cd No_RH/
bash wheat_tilling_pub/postprocessing/vcf_modifications/fixMAPSOutputAndMakeVCF.sh

mkdir RH
mv *RH_only* RH/ && cd RH
bash ./wheat_tilling_pub/postprocessing/vcf_modifications/fixMAPSOutputAndMakeVCF.sh
```

---
### 9. (MAPS) Summarizing Mutations Per Prameter

📥 Inputs   
• `all.mapspart2.Lib20HetMinPer15HetMinCov${j}HomMinCov${k}.reformatted.corrected.10kb_bins.RH.byContig.MI.No_RH.maps.vcf`: mutations **not from** residual hetrogenity regions

📥 Outputs    
• `Mutations.summary`: mutation numbers per sample per parameter  


The MAPS outputs were generated with four pairs of HomMinCov and HetMinCov in the previous step
```
from least to most stringency: HetMinCov3HomMinCov2, HetMinCov4HomMinCov3, HetMinCov5HomMinCov3 and HetMinCov6HomMinCov4
```

If any of the thresholds yielded ≥ 98% EMS rates, the following criteria are applied.
```
High confidence: the least stringent threshold yielding ≥ 98% EMS rates
Medium confidence: the least stringent threshold yielding ≥ 97% EMS rates among the remainning three, N/A otherwise
Low confidence: the least stringent threshold yielding ≥ 95% EMS rates among the remainning ones, N/A otherwise
```

If none of the thresholds yielded ≥ 98% EMS rates, pre-defined confidence levels will be used as [Krasileva et al., 2017](https://www.pnas.org/doi/10.1073/pnas.1619268114).
```
High confidence: HetMinCov5HomMinCov3
Medium confidence: HetMinCov4HomMinCov3
Low confidence: HetMinCov3HomMinCov2
```

Summarize the statistics and select the parameters. This will create Mutations. summary
```
#run this within the No_RH folder.
python summarize.py
```
---
### 10. (MAPS) Rescuing Multi-mapped Reads
This step goes back to **4. (MAPS) Sorting and Deduplication** and requires *.sorted.rmdup.bam files. Multi-mapped reads were rescored, so that any meaningful mutations could be picked up by the MAPS pipeline.

📥 Inputs   
• `*.sorted.rmdup.bam`: sorted alignment files without duplicate reads  

📥 Outputs    
• `*.sorted.rmdup.rescued.bam`: sorted alignment files without duplicate reads  

Note: Once the output files are generated, the entire MAPS pipeline (steps 5 - 9) needs to rerun on these files.
```
for bam in *.sorted.rmdup.bam; do
    prefix="${bam%.bam}"

    # Convert BAM to SAM and extract header
    samtools view -@56 -h "${bam}" > "${prefix}.sam"
    samtools view -H "${prefix}.sam" > header.sam

    # Run multi-map corrector
    python ./wheat_tilling_pub/postprocessing/multi_map/multi-map-corrector-V1.6.py -i "${prefix}.sam" -l broken_chr.length.list > "${prefix}.rescued.sam"

    # Merge header and body, then convert to BAM
    cat header.sam "${prefix}.rescued.sam" | samtools view -@ 56 -b -o "${prefix}.rescued.bam"

    # Clean up temporary files
    rm "${prefix}.sam" "${prefix}.rescued.sam" header.sam
done
```

---
### 11. (MAPS) Finalization

📥 Inputs   
• `TSVs/No_RH TSVs/RH`: all tsv files for uniquely mapped regions in non-RH and RH regions  
• `TSVs-Multi/No_RH TSVs-Multi/RH`: all tsv files for uniquely mapped regions in non-RH and RH regions  

```
python finalize_vcf.py
```

We will create four final outputs! 
```
# Excluding genomic regtions with residual hetrogenity. Indels only 
Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.No_RH.maps.indels.vcf
# Excluding genomic regtions with residual hetrogenity. Substitutions only ------- This is our primary output 
Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.No_RH.maps.substitutions.vcf
# Only genomic regtions with residual hetrogenity. Indels only 
Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.RH_only.maps.indels.vcf
# Only genomic regtions with residual hetrogenity. Substitutions only 
Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.RH_only.maps.substitutions.vcf
```

Within these vcf files, the following fields will be included. Our primary mutations are labeled with Substitution/False/High/Any threshold/Unique. 
```
Type={Substiution/Indel}: This indicates whether called mutations are substitutions or indels. 
RH={True/False}: This indicates whether called mutations come from regions associated with residual hetrogenity.  
Confidence={High/Medium/Low}: This indicates the confidence level of called mutations.
Threshold=HetMinCov{x}HomMinCov{y}: This indicates parameters used to call the mutations. 
Mapping={Unique/Multi}: This indicates whether called mutations come from uniquely mapped or multi-mapped reads. 
```

---
### 12. snpEff

📥 Inputs   
• `Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.*.vcf`: Final mutation files from the MAPS pipeline  

📥 Outputs   
• `Kronos_v1.1.Exom-capture.corrected.deduped.10kb_bins.RH.byContig.MI.*.snpeff.vcf`: Final mutation files from the MAPS pipeline with annotated mutation effects

```
for vcf in *.vcf:
    prefix="${vcf%.vcf}"
    java -jar snpEff.jar eff -v Kronosv21 ${vcf} > ${prefix}.snpeff.vcf
done
```

---
### The GATK pipeline

To call variants with GATK, we began with the trimmed filtered sequencing outputs from Step **2. Quality Control and Filtering**. 

---

### 3. (GATK) Read Alignment
📥 Inputs   
• `*.filtered.fastq`: filtered and trimmed paired-end fastq files   
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: Kronos reference genome v1.1 

📥 Outputs   
• `*.sam`: alignment files  

```
for read1 in *.1.filtered.fastq; do
  accession="${read1%.1.filtered.fastq}"
  read2="${accession}.2.filtered.fastq"

  #align to the broken genome
  bwa mem -t ${Numthreads} ${reference_dir}/Kronos "${read1}" "${read2}" > "${accession}.gatk.sam"
```
---

### 4. (GATK) Sorting and Deduplication

📥 Inputs   
• `*.gatk.sam`: alignment files


📥 Outputs   
• `*.gatk.sorted.rmdup.bam`: sorted alignment files without duplicate reads

```
  samtools view -@56 -h "${accession}.gatk.sam" | samtools sort -@56 -o "${accession}.gatk.sorted.bam"
  samtools index -@56 "${accession}.gatk.sorted.bam"
  picard MarkDuplicates REMOVE_DUPLICATES=true I="${accession}.gatk.sorted.bam" O="${accession}.gatk.sorted.rmdup.bam" M="${accession}.rmdup.txt"
```
---

### 5. (GATK) Merging Redundant Libraries
📥 Inputs  
• `*.gatk.sorted.rmdup.bam`: sorted alignment files without duplicate reads  
• `exome_merge.list`: a list of alignments to be merged together    

📥 Outputs  
• `*.gatk.sorted.rmdup.bam`: merged sorted alignment files without duplicate read  

```
while read mutant lib1 lib2; do
     samtools merge -@ 56 -o ../sorted.rmdup.bam_files/${lib1}.gatk.sorted.rmdup.bam ${lib1}.sorted.rmdup.bam ${lib2}.gatk.sorted.rmdup.bam
 done < exome_merge.list
```
---
### 6. (GAKT) HaplotypeCaller
Note: variant effect prediction should be followed after this step. Refer to **12. snpEff**.  

📥 Inputs  
• `*.gatk.sorted.rmdup.bam`: merged sorted alignment files without duplicate read  
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: Kronos reference genome v1.1    

📥 Outputs  
• `*.vcf.gz`: variant calling outputs    

```
for bam in *.gatk.sorted.rmdup.bam; do
  accession="${bam%.gatk.sorted.rmdup.bam}"
  gatk HaplotypeCaller -R Kronos.collapsed.chromosomes.masked.v1.1.broken.fa -I ${accession}.gatk.sorted.rmdup.bam -O ${accession}.vcf.gz
  gunzip ${accession}.vcf.gz
  python reformat_gatk.py ${accession}.vcf
done
```
---

## Promoter-capture Data Remapping
The analysis of promoter-capture sequencing data was done nearly identically as the exome capture data analysis. Thus, only the workflow different from the previous one is recorded below. 

---

### 1. Downloading Sequencing Data
📥 Inputs  
• `promoter_MAPS_groups.list`: SRA identifiers for exome-capture data.  
Note: We obtained most of the data directly from the authors. Any missing sequencing data were downloaded from PRJNA1218005 produced by [Zhang et al. (2023)](https://www.pnas.org/doi/10.1073/pnas.2306494120).  

---

### 2. Quality Control and Filtering

📥 Inputs  
• `*.fastq.gz`: paired-end fastq files  

📥 Outputs  
• `*.trimmed.fq.gz`: paired-end fastq files  

```
for fq1 in *R1*fastq.gz; do
        fq2=$(echo $fq1 | sed 's/R1/R2/g')
        out1=$(echo $fq1 | sed 's/.fastq.gz/_trimmed.fq.gz/g')
        out2=$(echo $fq2 | sed 's/.fastq.gz/_trimmed.fq.gz/g')

        if [[ ! -f "${out1}" ]]; then
                trimmomatic PE $fq1 $fq2 \
                             $out1 ${fq1}_trimmed_unpaired.fq.gz \
                             $out2 ${fq2}_trimmed_unpaired.fq.gz \
                             ILLUMINACLIP:TruSeq3-PE-2.fa:2:30:10:2:keepBothReads \
                             LEADING:3 TRAILING:3 MINLEN:36 SLIDINGWINDOW:4:20
        fi
done
```
---

### 5. (MAPS) Merging Redundant Libraries
After processing all sequencing data according to the batch information (see the next step), we picked one sequencing datasets per mutant. This information can be found in `promoter_merge.list`.

---
### 6. (MAPS) Preparing for MAPS Pipeline
The BioProject PRJNA1218005 contains 33 BioSamples. Each BioSample contains a batch of Kronos mutants to be processed together. This informtion can be found in `promoter_MAPS_group.list`.

---
### 7. (MAPS) Running the MAPS Pipeline
l was set as the number of analyzed mutatns in a batch * 0.8 rounded up to the closest integer.



## Analyses

### NLR detection in Kronos mutants

📥 Inputs     
• `Kronos_all.NLRs.318_target.list`: A list of low-quality NLRs annotated with "|Interrupted|1". Only primary transcripts (.1) were considered.  
• `Kronos.collapsed.chromosomes.masked.v1.1.broken.fa`: Kronos reference genome v1.1   
• `*.vcf`: vcf files for mutants (we used those from GATK).  
• `Wheat_NLR`: Pre-trained Augustus parameters. See **the NLR_anlyses folder**.  

📥 Outputs     
• `*.NLRs.gff`: predicted NLR sequences in target regions of Kronos mutants.  
• `rescued_nlrs.gff3`: rescued NLR sequences (see the **output** folder)

```
#define target region, extract and modify genomic sequences
bedtools slop -i Kronos_all.NLRs.318_target.list -r 1000 -l 1000 -g genome.idx | awk '$3 == "mRNA" {print}' > Kronos_all.NLRs.318_target.padded.bed
bedtools getfasta -fi Kronos.collapsed.chromosomes.masked.v1.1.fa -bed  Kronos_all.NLRs.318_target.padded.bed -name -fo nlr_regions.fa
sed -i 's/mRNA:://g' nlr_regions.fa
python modify_vcf_nlr_region.py ${chromosome} #produce a vcf file with shifted coordinates for target genomic regions
bgzip -f $vcf
bcftools index ${vcf}.gz
bcftools consensus -H A -f nlr_regions.fa -o ${mutant}.nlr_regions.fa -s ${mutant} ${vcf}.gz
```
```
#for each modified genomic sequene
augustus --species=Wheat_NLR --genemodel=complete --gff3=on --noInFrameStop=true ${fa} > ${fa}.NLRs.gff
gffread -y ${prefix}.nlrs.fa -g ${prefix}.fa ${fa}.NLRs.gff
```

### UMAP
📥 Inputs     
• `*.vcf`: GATK-derived vcf files for mutants.  

📥 Outputs   
• `mutant_cluster_colors.tsv`: the membership of each Kronos mutants based on the UMAP (see the **output** folder)
```
#evaluate zygosity for common sites
collect_common_mutations.py True 0.05 #exome-capture, EMS type mutations
collect_common_mutations.py False 0.05 #exome-capture, non EMS type mutations

#similary for promoter capture
collect_common_mutations.py True 0.15 #promoter-capture, EMS type mutations
collect_common_mutations.py False 0.15 #promoter-capture, non EMS type mutations
```

```
#then we used
common_mutations_UMAP.R
```

