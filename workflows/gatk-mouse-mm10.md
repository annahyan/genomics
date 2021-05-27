# Creating GATK mm10 resource bundle


The GATK resource bundle is a collection of standard files for working with human resequencing data.
It contains known SNPs and indels to be used for BaseRecalibrator, RealignerTargetCreator, and IndelRealigner.
This is an attempt to recreate a similar bundle for the mouse genome (UCSC build mm10).

For mouse SNPs, it's possible to use the dbSNP database, which should be comparable to the human version.

Download dbSNP GRCm38 VCF files (each chromosome is in a separate file):

```bash
wget --recursive --no-parent --no-directories \
--accept vcf*vcf.gz \
ftp://ftp.ncbi.nih.gov/snp/organisms/archive/mouse_10090/VCF/
```

Delete the non-primary chromosomes if they are not included in the reference FASTA file.

Add "chr" to each chromosome (convert from GRCm38 to mm10 format):

```bash
for vcf in $(ls -1 vcf_chr_*.vcf.gz) ; do
  vcf_new=${vcf/.vcf.gz/.vcf}
  echo $vcf
  zcat $vcf | sed 's/^\([0-9XY]\)/chr\1/' > $vcf_new
  rm -fv $vcf
done
```
Adding #contig lines to the header, so that GATK GatherVcfs can process them.

```bash
for vcf in $(ls -1 vcf_chr_*.vcf) ; do
  vcf_dict=${vcf%.vcf}"_dict.vcf"
  echo $vcf
  gatk UpdateVCFSequenceDictionary -V $vcf --source-dictionary genome.dict  --output $vcf_dict
  
  rm -fv $vcf
done
```

Combine all dbSNP VCF files into one:

```bash
# generate parameter string containing all VCF files
vcf_file_string=""
for vcf in $(ls -1 vcf_chr_*_dict.vcf) ; do
  vcf_file_string="$vcf_file_string -I $vcf"
done
echo $vcf_file_string > arguments_file #this didn't really work

# concatenate VCF files 
# gatk 4.0.1.2-java-1.8 version didn't really work. 4.2.0 worked instead
gatk GatherVcfs -I vcf_chr_10_dict.vcf -I vcf_chr_11_dict.vcf -I vcf_chr_12_dict.vcf -I vcf_chr_13_dict.vcf -I vcf_chr_14_dict.vcf -I vcf_chr_15_dict.vcf -I vcf_chr_16_dict.vcf -I vcf_chr_17_dict.vcf -I vcf_chr_18_dict.vcf -I vcf_chr_19_dict.vcf -I vcf_chr_1_dict.vcf -I vcf_chr_2_dict.vcf -I vcf_chr_3_dict.vcf -I vcf_chr_4_dict.vcf -I vcf_chr_5_dict.vcf -I vcf_chr_6_dict.vcf -I vcf_chr_7_dict.vcf -I vcf_chr_8_dict.vcf -I vcf_chr_9_dict.vcf -I vcf_chr_AltOnly_dict.vcf -I vcf_chr_MT_dict.vcf -I vcf_chr_Multi_dict.vcf -I vcf_chr_Un_dict.vcf -I vcf_chr_X_dict.vcf -I vcf_chr_Y_dict.vcf -O dbsnp.vcf 

```

More recent dbSNP releases include a merged `00-All.vcf.gz` file in addition to the separate chromosome files.
Although it will not need to be concatenated, it will likely still need to be adjusted for GATK compatibility.

For mouse indels, the Sanger Mouse Genetics Programme (Sanger MGP) is probably the best resource.

Download all MGP indels (5/2015 release):

```bash
wget ftp://ftp-mouse.sanger.ac.uk/REL-1505-SNPs_Indels/mgp.v5.merged.indels.dbSNP142.normed.vcf.gz \
-O mgp.v5.indels.vcf.gz
```

Filter for passing variants with chr added:

```bash
# adjust header
zcat mgp.v5.indels.vcf.gz | head -1000 | grep "^#" | cut -f 1-8 \
| grep -v "#contig" | grep -v "#source" \
> mgp.v5.indels.pass.chr.vcf
# keep only passing and adjust chromosome name
zcat mgp.v5.indels.vcf.gz | grep -v "^#" | cut -f 1-8 \
| grep -w "PASS" | sed 's/^\([0-9MXY]\)/chr\1/' \
>> mgp.v5.indels.pass.chr.vcf
```

Sort VCF (automatically generated index has to be deleted due to a known bug):

```bash
java -Xms16G -Xmx16G -jar ${PICARD_ROOT}/picard.jar SortVcf VERBOSITY=WARNING \
SD=genome.dict \
I=mgp.v5.indels.pass.chr.vcf \
O=mgp.v5.indels.pass.chr.sort.vcf
rm -fv mgp.v5.indels.pass.chr.sort.vcf.idx
gatk IndexFeatureFile -I cohort.vcf.gz
```

Additional info:

* [GATK Resource Bundle](https://software.broadinstitute.org/gatk/download/bundle)
* [What should I use as known variants/sites for running tool X?](http://gatkforums.broadinstitute.org/gatk/discussion/1247/what-should-i-use-as-known-variants-sites-for-running-tool-x)
