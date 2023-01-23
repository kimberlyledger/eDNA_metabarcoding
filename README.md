# eDNA metabarcoding

repo for generic eDNA sequence analysis workflow

1. upload entire sequencing run to rawdata directory on eDNA VM 

for example: scp Y:\ABL_Genetics\eDNA\MiSeq_output\20230111_gadidmockcomm_bristolbay.zip  kimberly.ledger@161.55.97.134:/genetics/edna/rawdata

2. unzip and copy all reads to a gadid folder in workdir 

for example:  cp /genetics/edna/rawdata/20230111_gadidmockcomm_bristolbay/* /genetics/edna/workdir/gadids/20230111

3. and then move reads into project-specific folders  

for example: mv /genetics/edna/rawdata/workdir/gadids/20230111/e01* /genetics/edna/workdir/bristolbay/20230111  
**note: it is helpful to use revalent sample IDs in MiSeq sample sheet for easy organizing of specific files** 

4. fastq files output my MiSeq have already been demultiplex'd so we can move onto removing primers from amplicon reads using dada2 cutadapt tool

* make folder for trimmed reads: (mkdir trimmed)
* if cutadapt is not already installed: (conda create -n cutadaptenv cutadapt)
* activate enivronment: (conda activate cutadaptenv) 

for example: 
MiDeca primer sequences 
MiDeca_F: GGACGATAAGACCCTATAAA
MiDeca_R: ACGCTGTTATCCCTAAAG

set DATA= to the directory with the rawdata files:   
DATA=/genetics/edna/workdir/crabs/20230111

below, the first set of () creates an array, containing n elements of the desired trimmed and unique names:   
NAMELIST=$(ls ${DATA} | sed 's/e*_L001.*//' | uniq)
echo "${NAMELIST}"

iterate over all elements in the above array:   
for i in ${NAMELIST}; do
   cutadapt --discard-untrimmed -g GGACGATAAGACCCTATAAA -G ACGCTGTTATCCCTAAAG -o trimmed/${i}_R1.fastq.gz -p trimmed/${i}_R2.fastq.gz "$DATA/${i}_L001_R1_001.fastq.gz" "$DATA/${i}_L001_R2_001.fastq.gz";
done

**note: this created trimmed.fastq files that i had to remove (rm trimmed*)**

5. Unzip those trimmed files for analysis in DADA2

for example: pigz -d trimmed/*.gz

6. Create a "filtered" folder within the trimmed folder. and an "outputs" folder within the filtered folder. 

7. Run sequence_filtering.Rmd for DADA2 processing of trimmed reads
*when running this code here are the only things that need to be customized:* 
- file path 
- filter parameters and truncate lengths  
- merged sequence length filter 

*your outputs folder will now have:*
- seqtab.csv = ASV table for taxonomic analysis
- myasvs.fasta = fasta file with all ASVs 
- ASVtable.csv = ASV table with ASV# label as column names 
- asv_id_table.csv = a table of the ASV# label and ASV sequence 

*can move files to local computer for looking at data in Geneious, Excel, etc...* 
from command center (not logged into the VM)
for example: scp kimberly.ledger@161.55.97.134:/genetics/edna/workdir/gadids/20230111/S1_ND1_529_789/trimmed/filtered/outputs/* Downloads

8. Run blastn on "myasvs.fasta" output
for example: 
[kimberly.ledger@akc0ss-vu-134 outputs]$ nohup blastn -db nt -query myasvs.fasta -perc_identity 96 -qcov_hsp_perc 100 -num_threads 10 -out blastnresults_out -outfmt '6 qseqid sseqid pident length mismatch gapopen qstart qend sstart send evalue bitscore sscinames staxids'

**may need to retype - and ' because copy-paste format throws things off**

9. Run taxonkit on blastn output 
cat blastnresults_out | taxonkit lineage -c -i 14 > blastn_tax.out
taxonkit reformat blastn_tax.out -i 16 > blastn_taxlineage.txt

10. Use blastn_taxonomy.Rmd to generate ASV id's from 'blastn_taxlineage.txt'

11. copy metadata file to the VM for additional analyses 
for example: scp Z:\Gadids\Gadid_metabarcoding_PCR\20230111_gadidmetadata.csv kimberly.ledger@161.55.97.134:/genetics/edna/workdir/gadids/20230111
