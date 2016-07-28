# Automated workflow for rna-seq with multiple samples
### with TOPHAT2 and Cufflinks
```
#!/bin/sh
```

## defining folder structure

###User must specify the path to the RNA-seq project folder
```
PROJECTFOLDER=/home/chao/rnaseq/experiment1/
```
###User must specify the name of each sample. i.e. for sample1.fastq.gz, sample2.fastq.gz and sample3.fastq.gz you could write: (this applies to paired-end read files also)
```
SAMPLES_ID="1 2 3"
```
## reference genome, index and annotation

###User to specify the path to these files, and define some parameters
```
GENE_REFERENCE=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Annotation/Genes/genes.gtf # path to reference annotation gtf or gff file
BOWTIE_INDEX=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Sequence/Bowtie2Index/human38 # path to bowtie2 index (for tophap2)
REFERENCEFA=/lib/GenomeRefs/Homo_sapiens/Ensembl/GRCh38/Sequence/WholeGenomeFasta/genome.fa # path to reference genome
P=22 # number of threads to use for data analysis (find out number of availible threads on linux using "htop" command)
LIBRARYTYPE=fr-firststrand # library type depends on your library-prep protocol.
```
## ALL done! Kick back with a mojito and let the computer take care of the rest.

###these folders below are auto-generated during running the script. Each stage of the processed data will be saved under one these folders.
```
SCRIPTS=$PROJECTFOLDER/scripts/
FASTQFILES=$PROJECTFOLDER/data/
ALIGNMENTFILES=$PROJECTFOLDER/alignments/
ASSEMBLIESFILES=$PROJECTFOLDER/assemblies/
QUANTIFICATION=$PROJECTFOLDER/quantification
DE=$PROJECTFOLDER/DE/
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
```
mkdir $DE/non_timeseries/

LABELS="$(for SAMPLE_ID in $SAMPLES;do echo "sample$SAMPLE_ID"|tr "\n" ",";done)"
QUANTFILES="$(for SAMPLE_ID in $SAMPLES;do echo "$QUANTIFICATION/sample$SAMPLE_ID/abundances.cxb"|tr "\n" " ";done)"

cat > $SCRIPTS/cuffdiff.sh <<EOF
cuffdiff -o $DE/test/ -L $LABELS -p $P -b $REFERENCEFA -u --library-type $LIBRARYTYPE $ASSEMBLIESFILES/merged/merged.gtf $QUANTFILES
EOF

bash $SCRIPTS/cuffdiff.sh

echo "DE completed"

```

###### Credit must be given to alyssafrazee@Github, for which a lot of this script is based on
