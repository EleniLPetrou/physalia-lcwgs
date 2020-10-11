Tutorial 1: data processing
================

  - [Initial preparations](#initial-preparations)
      - [Create a project directory](#create-a-project-directory)
      - [Create a sample list](#create-a-sample-list)
      - [Prepare a sample table](#prepare-a-sample-table)
  - [Data processing pipeline](#data-processing-pipeline)
      - [Examine the raw fastq files](#examine-the-raw-fastq-files)
      - [Adapter clipping](#adapter-clipping)
      - [Build reference index files](#build-reference-index-files)
      - [Map to reference, sort, and quality
        filter](#map-to-reference-sort-and-quality-filter)
      - [Merge samples that were sequenced multiple times and generate a
        merged sample table and bam
        list](#merge-samples-that-were-sequenced-multiple-times-and-generate-a-merged-sample-table-and-bam-list)
      - [Deduplicate (all samples) and clip overlapping read pairs
        (pair-end reads
        only)](#deduplicate-all-samples-and-clip-overlapping-read-pairs-pair-end-reads-only)
      - [In-del relignment](#in-del-relignment)
      - [Read count](#read-count)
      - [Summarize counting result](#summarize-counting-result)

In this tuturial, you will learn how to take the raw sequencing files in
`fastq` format, and convert them into alignment files in `bam` format.
Along the way, you will perform multiple quality control (QC)
procedures, and will map the short sequences to a reference genome.

## Initial preparations

#### Create a project directory

As a first step, you should create a project directory (referred to as
`BASEDIR` in certain scripts), with the following subdirectories:

  - `raw_fastq` raw fastq files

  - `adapter_clipped` adapter clipped fastq files

  - `bam` bam files

  - `sample_lists` sample tables, sample lists, and other small text
    files

  - `fastqc` FastQC output

  - `markdowns` RMarkdown and Markdown files

#### Create a sample list

A sample list is a list of the prefixes of raw fastq files. These should
be unique for each fastq file, and the rest of the names have to be the
same for all raw fastq files. No header should be included in this list.

Here is an example of a sample list:
<https://github.com/therkildsen-lab/greenland-cod/blob/master/sample_lists/sample_list_pe_1.tsv>

#### Prepare a sample table

A sample table is a **tab deliminated** table that includes relevant
information for all fastq files. It should include the following six
columns, strictly in this order:

  - `prefix` prefix of raw fastq files; it should be the union of all
    your sample lists

  - `lane_number` lane number; each sequencing lane should be assigned a
    different number

  - `seq_id` sequence ID，this can be the same thing as sample ID or lane
    ID and it does not matter except for when different libraries were
    prepared out of the same sample and were run in the same lane. In
    this case, seq\_id should be used to distinguish these.

  - `sample_id` sample ID

  - `population` population name

  - `data_type` data type; there can only be two possible entries: `pe`
    (for paired-end data) or `se` (for single end data)

It is important to make sure that the combination of lane\_number,
seq\_id, and sample\_id has to be unique for each fastq file.

An example of a sample table:
<https://github.com/therkildsen-lab/greenland-cod/blob/master/sample_lists/sample_table.tsv>

## Data processing pipeline

#### Examine the raw fastq files

###### fastq file structure

``` bash
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
RAWFASTQSUFFIX1=_Paired_NoCont_1.fastq.gz # Suffix to raw fastq files. Use forward reads with paired-end data.
for SAMPLE in `cat $SAMPLELIST`; do
  echo $SAMPLE
  zcat $BASEDIR'raw_fastq/'$SAMPLE$RAWFASTQSUFFIX1 | head -n 4
  echo ' '
done
```

###### FastQC report

``` bash
FASTQC=/programs/bin/fastqc/fastqc
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
RAWFASTQSUFFIX1=_Paired_NoCont_1.fastq.gz # Suffix to raw fastq files. Use forward reads with paired-end data.
RAWFASTQSUFFIX2=_Paired_NoCont_2.fastq.gz # Suffix to raw fastq files. Use reverse reads with paired-end data.

for SAMPLE in `cat $SAMPLELIST`; do
  $FASTQC $BASEDIR'raw_fastq/'$SAMPLE$RAWFASTQSUFFIX1 -o $BASEDIR'fastqc/'
  $FASTQC $BASEDIR'raw_fastq/'$SAMPLE$RAWFASTQSUFFIX2 -o $BASEDIR'fastqc/'
done
```

#### Adapter clipping

``` bash
TRIMMOMATIC=/programs/trimmomatic/trimmomatic-0.39.jar
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table.tsv # Path to a sample table where the 1st column is the prefix of the raw fastq files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The combination of these three columns have to be unique. The 6th column should be data type, which is either pe or se. 
RAWFASTQDIR=$BASEDIR/raw_fastq/ # Path to raw fastq files. 
RAWFASTQSUFFIX1=_Paired_NoCont_1.fastq.gz # Suffix to raw fastq files. Use forward reads with paired-end data.
RAWFASTQSUFFIX2=_Paired_NoCont_2.fastq.gz # Suffix to raw fastq files. Use reverse reads with paired-end data. 
ADAPTERS=$BASEDIR/reference/NexteraPE_NT.fa # Path to a list of adapter/index sequences.

## Loop over each sample
for SAMPLEFILE in `cat $SAMPLELIST`; do
    
    ## Extract relevant values from a table of sample, sequencing, and lane ID (here in columns 4, 3, 2, respectively) for each sequenced library
    SAMPLE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 4`
    SEQ_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 3`
    LANE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 2`
    SAMPLE_SEQ_ID=$SAMPLE_ID'_'$SEQ_ID'_'$LANE_ID  # When a sample has been sequenced in multiple lanes, we need to be able to identify the files from each run uniquely
    
    ## Extract data type from the sample table
    DATATYPE=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 6`
    
    ## The input and output path and file prefix
    RAWFASTQ_ID=$RAWFASTQDIR$SAMPLEFILE
    SAMPLEADAPT=$BASEDIR'adapter_clipped/'$SAMPLE_SEQ_ID
    
    ## Adapter clip the reads with Trimmomatic
    # The options for ILLUMINACLIP are: ILLUMINACLIP:<fastaWithAdaptersEtc>:<seed mismatches>:<palindrome clip threshold>:<simple clip threshold>:<minAdapterLength>:<keepBothReads>
    # For definitions of these options, see http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf
    if [ $DATATYPE = pe ]; then
        java -jar $TRIMMOMATIC PE -threads 18 -phred33 $RAWFASTQ_ID$RAWFASTQSUFFIX1 $RAWFASTQ_ID$RAWFASTQSUFFIX2 $SAMPLEADAPT'_adapter_clipped_f_paired.fastq.gz' $SAMPLEADAPT'_adapter_clipped_f_unpaired.fastq.gz' $SAMPLEADAPT'_adapter_clipped_r_paired.fastq.gz' $SAMPLEADAPT'_adapter_clipped_r_unpaired.fastq.gz' 'ILLUMINACLIP:'$ADAPTERS':2:30:10:1:true'
    
    elif [ $DATATYPE = se ]; then
        java -jar $TRIMMOMATIC SE -threads 18 -phred33 $RAWFASTQ_ID$RAWFASTQSUFFIX1 $SAMPLEADAPT'_adapter_clipped_se.fastq.gz' 'ILLUMINACLIP:'$ADAPTERS':2:30:10'
    fi
    
done
```

#### Build reference index files

``` bash
PICARD=/programs/picard-tools-2.19.2/picard.jar
SAMTOOLS=/programs/bin/samtools/samtools
BOWTIEBUILD=/programs/bin/bowtie2/bowtie2-build
BASEDIR=/workdir/physalia-lcwgs/day_1/
REFERENCE=$BASEDIR/reference/mme_physalia_testdata_chr24.fa
REFBASENAME="${REFERENCE%.*}"
$SAMTOOLS faidx $REFERENCE
java -jar $PICARD CreateSequenceDictionary R=$REFERENCE O=$REFBASENAME'.dict'
$BOWTIEBUILD $REFERENCE $REFBASENAME
```

#### Map to reference, sort, and quality filter

``` bash
BOWTIE=/programs/bin/bowtie2/bowtie2
SAMTOOLS=/programs/bin/samtools/samtools
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table.tsv # Path to a sample table where the 1st column is the prefix of the raw fastq files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The combination of these three columns have to be unique. The 6th column should be data type, which is either pe or se. 
FASTQDIR=$BASEDIR/adapter_clipped/ # Path to the directory where fastq file are stored. 
FASTQSUFFIX1=_adapter_clipped_f_paired.fastq.gz # Suffix to fastq files. Use forward reads with paired-end data. 
FASTQSUFFIX2=_adapter_clipped_r_paired.fastq.gz # Suffix to fastq files. Use reverse reads with paired-end data. 
MAPPINGPRESET=very-sensitive # The pre-set option to use for mapping in bowtie2 (very-sensitive for end-to-end (global) mapping [typically used when we have a full genome reference], very-sensitive-local for partial read mapping that allows soft-clipping [typically used when mapping genomic reads to a transcriptome]
REFERENCE=$BASEDIR/reference/mme_physalia_testdata_chr24.fa # Path to reference fasta file and file name
REFNAME=mme_physalia_testdata_chr24 # Reference name to add to output files, e.g. gadMor2

## Loop over each sample
for SAMPLEFILE in `cat $SAMPLELIST`; do
    
    ## Extract relevant values from a table of sample, sequencing, and lane ID (here in columns 4, 3, 2, respectively) for each sequenced library
    SAMPLE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 4`
    SEQ_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 3`
    LANE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 2`
    SAMPLE_SEQ_ID=$SAMPLE_ID'_'$SEQ_ID'_'$LANE_ID
    
    ## Extract data type from the sample table
    DATATYPE=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 6`
    
    ## The input and output path and file prefix
    SAMPLETOMAP=$FASTQDIR$SAMPLE_SEQ_ID
    SAMPLEBAM=$BASEDIR'bam/'$SAMPLE_SEQ_ID
    
    ## Define platform unit (PU), which is the lane number
    PU=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 2`
    
    ## Define reference base name
    REFBASENAME="${REFERENCE%.*}"
    
    ## Map reads to the reference 
    # Map the paired-end reads
    if [ $DATATYPE = pe ]; then 
    # We ignore the reads that get orphaned during adapter clipping because that is typically a very small proportion of reads. If a large proportion of reads get orphaned (loose their mate so they become single-end), these can be mapped in a separate step and the resulting bam files merged with the paired-end mapped reads.
    $BOWTIE -q --phred33 --$MAPPINGPRESET -p 16 -I 0 -X 1500 --fr --rg-id $SAMPLE_SEQ_ID --rg SM:$SAMPLE_ID --rg LB:$SAMPLE_ID --rg PU:$PU --rg PL:ILLUMINA -x $REFBASENAME -1 $SAMPLETOMAP$FASTQSUFFIX1 -2 $SAMPLETOMAP$FASTQSUFFIX2 -S $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.sam'
    
    # Map the single-end reads
    elif [ $DATATYPE = se ]; then
    $BOWTIE -q --phred33 --$MAPPINGPRESET -p 16 --rg-id $SAMPLE_SEQ_ID --rg SM:$SAMPLE_ID --rg LB:$SAMPLE_ID --rg PU:$PU --rg PL:ILLUMINA -x $REFBASENAME -U $SAMPLETOMAP$FASTQSUFFIX1 -S $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.sam'
    
    fi
    
    ## Convert to bam file for storage
    $SAMTOOLS view -bS -F 4 $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.sam' > $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.bam'
    rm $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.sam'
    
    ## Filter the mapped reads
    # Filter bam files to remove poorly mapped reads (non-unique mappings and mappings with a quality score < 20) -- do we want the quality score filter??
    $SAMTOOLS view -h -q 20 $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'.bam' | samtools view -buS - | samtools sort -o $SAMPLEBAM'_'$DATATYPE'_bt2_'$REFNAME'_minq20_sorted.bam'
    
done
```

#### Merge samples that were sequenced multiple times and generate a merged sample table and bam list

We need a new sample table, because prior to merging, each row represent
a fastq file, but after merging, each row represent a unique sample. The
merged sample table has a slightly different formatting.

In this example, we don’t have any samples that were sequenced multiple
times, but we still need to generate a new sample table with different
formatting.

``` r
library(tidyverse)
## Define base directory and reference name
basedir <- "/workdir/physalia-lcwgs/day_1/"
refname <- "mme_physalia_testdata_chr24"
## Read in unmerged sample tables and combine pe and se
sample_table_merged <- read_tsv(paste0("../sample_lists/sample_table.tsv")) %>%
  mutate(sample_seq_id=paste(sample_id,seq_id,lane_number, data_type, sep = "_")) %>%
  select(sample_seq_id, lane_number, seq_id, sample_id, population, data_type)
write_tsv(sample_table_merged, paste0("../sample_lists/sample_table_merged.tsv"))
bam_list_merged <- paste0(basedir, "bam/", sample_table_merged$sample_seq_id, "_bt2_", refname, "_minq20_sorted.bam")
bam_list_dedup_overlapclipped <- transmute(sample_table_merged, suffix=ifelse(data_type=="se", paste0("_bt2_", refname, "_minq20_sorted_dedup.bam"), paste0("_bt2_", refname, "_minq20_sorted_dedup_overlapclipped.bam"))) %>%
  .$suffix %>%
  paste0(basedir, "bam/", sample_table_merged$sample_seq_id, .)
bam_list_realigned <- transmute(sample_table_merged, suffix=ifelse(data_type=="se", paste0("_bt2_", refname, "_minq20_sorted_dedup_realigned.bam"), paste0("_bt2_", refname, "_minq20_sorted_dedup_overlapclipped_realigned.bam"))) %>%
  .$suffix %>%
  paste0(basedir, "bam/", sample_table_merged$sample_seq_id, .)
write_lines(bam_list_merged, paste0("../sample_lists/bam_list_merged.txt"))
write_lines(bam_list_dedup_overlapclipped, paste0("../sample_lists/bam_list_dedup_overlapclipped.txt"))
write_lines(bam_list_realigned, paste0("../sample_lists/bam_list_realigned.txt"))
```

#### Deduplicate (all samples) and clip overlapping read pairs (pair-end reads only)

``` bash
PICARD=/programs/picard-tools-2.19.2/picard.jar
BAMUTIL=/programs/bamUtil/bam
BASEDIR=/workdir/physalia-lcwgs/day_1/
BAMLIST=$BASEDIR/sample_lists/bam_list_merged.txt # Path to a list of merged bam files.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table_merged.tsv # Path to a sample table where the 1st column is the prefix of the MERGED bam files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The 5th column is population name and 6th column is the data type.
REFNAME=mme_physalia_testdata_chr24 # Reference name to add to output files

## Loop over each sample
for SAMPLEBAM in `cat $BAMLIST`; do
    
    ## Extract the file name prefix for this sample
    SAMPLEPREFIX=`echo $SAMPLEBAM | sed 's/_bt2_.*//' | sed -e 's#.*/bam/\(\)#\1#'`
    
    ## Remove duplicates and print dupstat file
    # We used to be able to just specify picard.jar on the CBSU server, but now we need to specify the path and version
    java -Xmx60g -jar $PICARD MarkDuplicates I=$SAMPLEBAM O=$BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dedup.bam' M=$BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dupstat.txt' VALIDATION_STRINGENCY=SILENT REMOVE_DUPLICATES=true
    
    ## Extract data type from the merged sample table
    DATATYPE=`grep -P "${SAMPLEPREFIX}\t" $SAMPLETABLE | cut -f 6`
    
    if [ $DATATYPE != se ]; then
        ## Clip overlapping paired end reads (only necessary for paired end data)
        $BAMUTIL clipOverlap --in $BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dedup.bam' --out $BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dedup_overlapclipped.bam' --stats
    fi
    
done
```

#### In-del relignment

``` bash
## Use an older version of Java
#export JAVA_HOME=/usr/local/jdk1.8.0_121
#export PATH=$JAVA_HOME/bin:$PATH

JAVA=/usr/local/jdk1.8.0_121/bin/java
SAMTOOLS=/programs/bin/samtools/samtools
GATK=/programs/GenomeAnalysisTK-3.7/GenomeAnalysisTK.jar
BASEDIR=/workdir/physalia-lcwgs/day_1/

cp $BASEDIR/sample_lists/bam_list_dedup_overlapclipped.txt $BASEDIR/sample_lists/bam_list_dedup_overlapclipped.list

BAMLIST=$BASEDIR/sample_lists/bam_list_dedup_overlapclipped.list # Path to a list of merged, deduplicated, and overlap clipped bam files. Full paths should be included. This file has to have a suffix of ".list"
REFERENCE=$BASEDIR/reference/mme_physalia_testdata_chr24.fa # Path to reference fasta file and file name
REFNAME=mme_physalia_testdata_chr24 # Reference name to add to output files

## Loop over each sample
cd $BASEDIR/bam/
for SAMPLEBAM in `cat $BAMLIST`; do

if [ -e $SAMPLEBAM'.bai' ]; then
    echo "the file already exists"
else
    ## Index bam files
    $SAMTOOLS index $SAMPLEBAM
fi

done

## Realign around in-dels
# This is done across all samples at once

## Create list of potential in-dels
if [ ! -f $BASEDIR'bam/all_samples_for_indel_realigner.intervals' ]; then
  $JAVA -Xmx40g -jar $GATK \
  -T RealignerTargetCreator \
  -R $REFERENCE \
  -I $BAMLIST \
  -o $BASEDIR'bam/all_samples_for_indel_realigner.intervals' \
  -drf BadMate
fi

## Run the indel realigner tool
$JAVA -Xmx40g -jar $GATK \
-T IndelRealigner \
-R $REFERENCE \
-I $BAMLIST \
-targetIntervals $BASEDIR'bam/all_samples_for_indel_realigner.intervals' \
--consensusDeterminationModel USE_READS  \
--nWayOut _realigned.bam
```

#### Read count

##### Count fastq files

``` bash
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table.tsv # Path to a sample table where the 1st column is the prefix of the raw fastq files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The combination of these three columns have to be unique. The 6th column should be data type, which is either pe or se. 
RAWFASTQDIR=$BASEDIR/raw_fastq/ # Path to raw fastq files. 
SEQUENCER=@HWI # Sequencer name that appears in the beginning of the first line in a fastq file. 
QUALFILTERED=false # Whether the sample has gone through quality filtering. true or false
OUT=$BASEDIR/sample_lists/fastq_count.tsv

# Create headers for the output
if $QUALFILTERED; then
    printf 'sample_seq_id\traw_reads\traw_bases\tadapter_clipped_bases\tqual_filtered_bases\n' > $OUT
else
    printf 'sample_seq_id\traw_reads\traw_bases\tadapter_clipped_bases\n' > $OUT
fi

# Loop over each sample in the sample table
for SAMPLEFILE in `cat $SAMPLELIST`; do
  RAWFASTQFILES=$RAWFASTQDIR$SAMPLEFILE'*.gz'  # The input path and file prefix
  
  # Count the number of reads in raw fastq files. We only need to count the forward reads, since the reverse will contain exactly   the same number of reads. fastq files contain 4 lines per read, so the number of total reads will be half of this line number. 
  RAWREADS=`zcat $RAWFASTQFILES | wc -l`
  
  # Count the number of bases in raw fastq files. We only need to count the forward reads, since the reverse will contain exactly   the same number of bases. The total number of reads will be twice this count. 
  RAWBASES=`zcat $RAWFASTQFILES | grep -A 1 -E "^$SEQUENCER" | grep "^[ACGTN]" | tr -d "\n" | wc -m` 
  
  # Extract relevant values from a table of sample, sequencing, and lane ID (here in columns 4, 3, 2, respectively) for each sequenced library
  SAMPLE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 4`
  SEQ_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 3`
  LANE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 2`
  SAMPLE_SEQ_ID=$SAMPLE_ID'_'$SEQ_ID'_'$LANE_ID
  
  # Find all adapter clipped fastq files corresponding to this sample and store them in the object ADAPTERFILES.
  ADAPTERFILES=$BASEDIR'adapter_clipped/'$SAMPLE_SEQ_ID'*.gz'
  
  # Count all bases in adapter clipped files. 
  ADPTERCLIPBASES=`zcat $ADAPTERFILES | grep -A 1 -E "^$SEQUENCER" | grep "^[ACGTN]" | tr -d "\n" | wc -m`
  
  # If reads are quality filtered, count quality filtered files.
  if $QUALFILTERED; then
    # Find all quality trimmed fastq files corresponding to this sample and store them in the object QUALFILES.
    QUALFILES=$BASEDIR'qual_filtered/'$SAMPLE_SEQ_ID'*.gz'
    # Count bases in quality trimmed files.
    QUALFILTPBASES=`zcat $QUALFILES | grep -A 1 -E "^$SEQUENCER" | grep "^[ACGTN]" | tr -d "\n" | wc -m`
    # Write the counts in appropriate order.
    printf "%s\t%s\t%s\t%s\t%s\n" $SAMPLE_SEQ_ID $((RAWREADS/4)) $RAWBASES $ADPTERCLIPBASES $QUALFILTPBASES >> $OUT
    # When reads are not quality filtered, directly write the output
  
  else
    # Write the counts in appropriate order.
    printf "%s\t%s\t%s\t%s\n" $SAMPLE_SEQ_ID $((RAWREADS/4)) $RAWBASES $ADPTERCLIPBASES >> $OUT
  fi
done
```

##### Count unmerged bam files

``` bash
BASEDIR=/workdir/physalia-lcwgs/day_1/
SAMPLELIST=$BASEDIR/sample_lists/sample_list.txt # Path to a list of prefixes of the raw fastq files. It should be a subset of the the 1st column of the sample table.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table.tsv # Path to a sample table where the 1st column is the prefix of the raw fastq files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The combination of these three columns have to be unique. The 6th column should be data type, which is either pe or se. 
REFNAME=mme_physalia_testdata_chr24 # Reference name to add to output files
OUT=$BASEDIR/sample_lists/bam_count_unmerged.tsv

printf 'sample_seq_id\tmapped_bases\tqual_filtered_mapped_bases\n' > $OUT

for SAMPLEFILE in `cat $SAMPLELIST`; do
    
    # Extract relevant values from a table of sample, sequencing, and lane ID (here in columns 4, 3, 2, respectively) for each sequenced library
    SAMPLE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 4`
    SEQ_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 3`
    LANE_ID=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 2`
    SAMPLE_SEQ_ID=$SAMPLE_ID'_'$SEQ_ID'_'$LANE_ID
    
    ## Extract data type from the sample table
    DATATYPE=`grep -P "${SAMPLEFILE}\t" $SAMPLETABLE | cut -f 6`
    
    ## Count raw mapped bases
    RAWBAMFILE=$BASEDIR'bam/'$SAMPLE_SEQ_ID'_'$DATATYPE'_bt2_'$REFNAME'.bam'
    MAPPEDBASES=`samtools stats $RAWBAMFILE | grep ^SN | cut -f 2- | grep "^bases mapped (cigar)" | cut -f 2`
    
    ## Count quality filtered mapped bases
    QUALFILTBAMFILE=$BASEDIR'bam/'$SAMPLE_SEQ_ID'_'$DATATYPE'_bt2_'$REFNAME'_minq20_sorted.bam'
    QUAFILTBASES=`samtools stats $QUALFILTBAMFILE | grep ^SN | cut -f 2- | grep "^bases mapped (cigar)" | cut -f 2`
    
    printf "%s\t%s\t%s\n" $SAMPLE_SEQ_ID $MAPPEDBASES $QUAFILTBASES >> $OUT
    
done
```

##### Count merged bam files

``` bash
BASEDIR=/workdir/physalia-lcwgs/day_1/
BAMLIST=$BASEDIR/sample_lists/bam_list_merged.txt # Path to a list of merged bam files.
SAMPLETABLE=$BASEDIR/sample_lists/sample_table_merged.tsv # Path to a sample table where the 1st column is the prefix of the MERGED bam files. The 4th column is the sample ID, the 2nd column is the lane number, and the 3rd column is sequence ID. The 5th column is population name and 6th column is the data type.
REFNAME=mme_physalia_testdata_chr24 # Reference name to add to output files
OUT=$BASEDIR/sample_lists/bam_count_merged.tsv

printf 'sample_seq_id\tdedup_mapped_bases\tavg_fragment_size\toverlap_clipped_bases\n' > $OUT

for SAMPLEBAM in `cat $BAMLIST`; do
  
  ## Extract the file name prefix for this sample
  SAMPLEPREFIX=`echo $SAMPLEBAM | sed 's/_bt2_.*//' | sed -e 's#.*/bam/\(\)#\1#'`
  
  ## Count deduplicated bases
  DEDUPFILE=$BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dedup.bam'
  DEDUPMAPPEDBASES=`samtools stats $DEDUPFILE | grep ^SN | cut -f 2- | grep "^bases mapped (cigar)" | cut -f 2`
  
  ## Extract data type from the merged sample table
  DATATYPE=`grep -P "${SAMPLEPREFIX}\t" $SAMPLETABLE | cut -f 6`
  
  if [ $DATATYPE != se ]; then
    ## Calculate average fragment length for paired end reads
    AVGFRAG=`samtools view $DEDUPFILE | grep YT:Z:CP | awk '{sum+=sqrt($9^2)} END {printf "%f", sum/NR}'`
  if [ "$AVGFRAG" == '' ]; then AVGFRAG=0 ; fi
    
    ## Count overlap clipped bam files for paired end reads 
    CLIPOVERLAPFILE=$BASEDIR'bam/'$SAMPLEPREFIX'_bt2_'$REFNAME'_minq20_sorted_dedup_overlapclipped.bam'
    CLIPOVERLAPBASES=`samtools stats $CLIPOVERLAPFILE | grep ^SN | cut -f 2- | grep "^bases mapped (cigar)" | cut -f 2`
    
  else
    AVGFRAG=NA
    CLIPOVERLAPBASES=NA
  fi
printf "%s\t%s\t%s\t%s\n" $SAMPLEPREFIX $DEDUPMAPPEDBASES $AVGFRAG $CLIPOVERLAPBASES >> $OUT
done
```

#### Summarize counting result

``` r
library(tidyverse)
library(cowplot)
library(knitr)
fastq_count <- read_tsv("../sample_lists/fastq_count.tsv") %>% 
  mutate(sample_id=str_sub(sample_seq_id, 1, 13)) %>% 
  dplyr::select(-sample_seq_id)
bam_count_unmerged <- read_tsv("../sample_lists/bam_count_unmerged.tsv") %>% 
  mutate(sample_id=str_sub(sample_seq_id, 1, 13)) %>% 
  dplyr::select(-sample_seq_id)
bam_count_merged <- read_tsv("../sample_lists/bam_count_merged.tsv") %>% 
  mutate(sample_id=str_sub(sample_seq_id, 1, 13)) %>% 
  dplyr::select(-sample_seq_id)
count_final <- left_join(fastq_count, bam_count_unmerged, by="sample_id") %>%
  left_join(bam_count_merged, by="sample_id") %>%
  relocate(sample_id) %>%
  relocate(-avg_fragment_size)
count_final %>% kable()
```

| sample\_id     | raw\_reads | raw\_bases | adapter\_clipped\_bases | mapped\_bases | qual\_filtered\_mapped\_bases | dedup\_mapped\_bases | overlap\_clipped\_bases | avg\_fragment\_size |
| :------------- | ---------: | ---------: | ----------------------: | ------------: | ----------------------------: | -------------------: | ----------------------: | ------------------: |
| JekyllIs\_1068 |     298348 |   36997321 |                36988174 |      26141844 |                       4978495 |              4543651 |                 4482596 |            407.2688 |
| JekyllIs\_1072 |     255902 |   31764706 |                31758277 |      22872202 |                       4024791 |              3695336 |                 3651333 |            421.3267 |
| JekyllIs\_1073 |     262822 |   32571001 |                32546361 |      22685176 |                       4509380 |              4116072 |                 4006672 |            351.3786 |
| JekyllIs\_1074 |     203824 |   25247560 |                25241390 |      17649859 |                       3553748 |              3287001 |                 3247379 |            391.9289 |
| JekyllIs\_1034 |     223282 |   27707817 |                27701392 |      19716454 |                       3880819 |              3588796 |                 3544435 |            479.8185 |

``` r
count_final %>% 
  pivot_longer(cols = 3:8, names_to = "step", values_to = "base_count") %>%
  arrange(sample_id, desc(base_count)) %>%
  mutate(step=fct_reorder(step, base_count, mean, .desc=T)) %>%
  ggplot(aes(x=sample_id, y=base_count/2/10^6, fill=step)) +
  geom_col(position="identity", col="black") +
  ylab("average coverage") +
  coord_flip() +
  theme_cowplot()
```

![](data_processing_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->