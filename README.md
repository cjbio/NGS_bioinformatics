# Automated workflow for rna-seq with multiple samples
### with TOPHAT2 and Cufflinks
```
#!/bin/sh
```

## defining folder structure

###User must specify the path to the RNA-seq project folder
```
PROJECTFOLDER=/home/chao/rnaseq/youngvsold_MSC_0014_0017_190716/
```
###User must specify the name of each sample. i.e. for sample1.fastq.gz and sample2.fastq.gz, write:
```
SAMPLES_ID="sample1 sample2"
```
###these folders are auto-generated during running the script. Each stage of the processed data will be saved under one these folders. No user intervention needed.
```
SCRIPTS=$PROJECTFOLDER/scripts/
FASTQFILES=$PROJECTFOLDER/data/
ALIGNMENTFILES=$PROJECTFOLDER/alignments/
ASSEMBLIESFILES=$PROJECTFOLDER/assemblies/
QUANTIFICATION=$PROJECTFOLDER/quantification
DE=$PROJECTFOLDER/DE/
```

## reference genome, index and annotation

###User to specify the path to these files
```
GENE_REFERENCE=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Annotation/Genes/genes.gtf # path to reference annotation gtf or gff file
BOWTIE_INDEX=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Sequence/Bowtie2Index/human38 # path to bowtie2 index (for tophap2)
REFERENCEFA=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Sequence/WholeGenomeFasta/genome.fa # path to reference genome
P=22 # number of threads to use for data analysis (find out number of availible threads on linux using "htop" command)
LIBRARYTYPE=fr-firststrand
```

## TOPHAT2 alignment
```
mkdir $SCRIPTS $ALIGNMENTFILES $ASSEMBLIESFILES $QUANTIFICATION $DE
for SAMPLE_ID in $SAMPLES
do
cat > $SCRIPTS/tophat_${SAMPLE_ID}.sh <<EOF
mkdir $ALIGNMENTFILES/sample${SAMPLE_ID} &&
tophat -G $GENE_REFERENCE -p $P --library-type $LIBRARYTYPE -o $ALIGNMENTFILES/sample${SAMPLE_ID} $BOWTIE_INDEX $FASTQFILES/${SAMPLE_ID}_R1.fastq.gz $FASTQFILES/${SAMPLE_ID}_R2.fastq.gz &&
mv $ALIGNMENTFILES/sample${SAMPLE_ID}/accepted_hits.bam $ALIGNMENTFILES/sample${SAMPLE_ID}.bam
EOF

bash $SCRIPTS/tophat_${SAMPLE_ID}.sh

done &&

echo "alignment completed"
```
## cufflinks transcript assembly
```
for SAMPLE_ID in $SAMPLES
do
cat > $SCRIPTS/cufflinks_${SAMPLE_ID}.sh <<EOF
mkdir $ASSEMBLIESFILES/sample${SAMPLE_ID} &&
cufflinks -p $P -g $ALIGNMENTFILES --library-type $LIBRARYTYPE -o $ASSEMBLIESFILES/sample${SAMPLE_ID} $ALIGNMENTFILES/sample${SAMPLE_ID}.bam &&
mv $ASSEMBLIESFILES/sample${SAMPLE_ID}/transcripts.gtf $ASSEMBLIESFILES/sample${SAMPLE_ID}_transcripts.gtf
EOF

bash $SCRIPTS/cufflinks_${SAMPLE_ID}.sh

done &&

echo "transcript assembly completed"
```
## merging multiple transcripts
```
for SAMPLE_ID in $SAMPLES
do 
echo "$ASSEMBLIESFILES/sample${SAMPLE_ID}_transcripts.gtf" >> $ASSEMBLIESFILES/assemblie_file_path.txt
done &&

mkdir $ASSEMBLIESFILES/merged/
cuffmerge -s $GENE_REFERENCE -p $P -s $REFERENCEFA -g $GENE_REFERENCE -o $ASSEMBLIESFILES/merged/ $ASSEMBLIESFILES/assemblie_file_path.txt &&

echo "merging transcripts completed"
```
## quantification of gene expression
```
for SAMPLE_ID in $SAMPLES
do
cat > $SCRIPTS/cuffquant_${SAMPLE_ID}.sh <<EOF
mkdir $QUANTIFICATION/sample${SAMPLE_ID} &&
cuffquant -o $QUANTIFICATION/sample${SAMPLE_ID}/ --library-type $LIBRARYTYPE -p $P $ASSEMBLIESFILES/merged/merged.gtf $ALIGNMENTFILES/sample${SAMPLE_ID}.bam
EOF

bash $SCRIPTS/cuffquant_${SAMPLE_ID}.sh

done &&

echo "quantification completed"
```
## differential gene expression
###After -L, user to specify the lable of each sample
###In the same order as the label, user to specify the path to cxb file (from quantification step) of each sample.
```
mkdir $DE/non_timeseries/
cuffdiff -o $DE/non_timeseries/ -L sample1,sample2 -p $P -b $REFERENCEFA -u --library-type $LIBRARYTYPE $ASSEMBLIESFILES/merged/merged.gtf $QUANTIFICATION/sampley/abundances.cxb $QUANTIFICATION/sample0/abundances.cxb

echo "DE completed"
```

###### Credit must be given to alyssafrazee@Github, for which a lot of this script is based on
