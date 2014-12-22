PhageHosts
==========

The code used for identification of the hosts of different phages. This is the complete code base used in Edwards, McNair, Wellington-Oguri, and Dutilh, manuscript in preparation. 

Almost all of the code is written in python and should work with version 2.7. Some parts of the code require matplotlib, numpy, and/or scipy. Some parts of the code were written in Perl 5. The code was written on CentOS 6 machines. Parts of the code include reference to running things on our cluster where we use SGE as the job scheduler. You can run those parts without the cluster, but it will probably be slower!


0. Datasets
===========

The phage genomes were downloaded from genbank and refseq and parsed to get the information in the "host" field. 

Use `python PhageHost/combine_gbff_fna.py viral.1.genomic.gbff viral.1.1.genomic.fna > viral.1.cds.fna`
to convert from GBFF to FNA format for all of the open reading frames

There are 1046 phages in genbank that have a host annotation:
`perl -ne '@a=split /\t/; print if ($a[2])' genbank.txt | grep YES$ | grep -i 'complete genome' | wc -l`

There are 971 phages in refseq that have a host annotation:
`PERL -ne '@a=split /\t/; print if ($a[2])' refseq.txt | grep YES$ | grep -i 'complete genome' | wc -l`

We use the refseq phages because we are going to get the refseq genomes.
use `perl get_viral_dna.pl`  to split the viruses into either phage or eukaryotic viruses. Obv we need phage for this!

Make a table of the heirarchy of the phage hosts. We are just going to keep these ranks: species, genus, family, order, class, phylum, superkingdom

I wrote code to automatically get the taxonomy for most of the hosts, but there were a few that I couldn't map, so I added those manually. To whit:

	'Acinetobacter genomosp.' : '471',
	'Actinobacillus actinomycetemcomitans' : '714',
	'alpha proteobacterium' : '34025',
	'Bacillus clarkii' : '79879',
	'Brevibacterium flavum' : '92706',
	'Celeribacter sp.' : '875171',
	'Escherichia sp.' : '237777',
	'Geobacillus sp.' : '340407',
	'Gordonia rubropertincta' : '36822',
	'Iodobacter sp.' : '641420',
	'Listeria sp.' : '592375',
	'Marinomonas sp.' : '127794',
	'Methanobacterium thermoautotrophicum' : '145262',
	'methicillin-resistant Staphylococcus' : '1280',
	'Nitrincola sp.' : '459834',
	'Persicivirga sp.' : '859306',
	'Salisaeta sp.' : '1392396',
	'Sulfitobacter sp.' : '191468'


Then you can generate the tsv file:
`python PhageHosts/code/phage_host_taxonomy.py  > phage_host_taxonomy.tsv`.

I also just made two files with tuples of genome NC id and taxonomy id.

###NOTE: For the phage, the tax id is the id of the HOST not of the phage###
This is important to remember (and I keep forgetting) but what we actually want to compare is the known host of a phage with the predicted host of the phage, so we need a list of taxonomy IDs of the phage host and the bacteria.

`python PhageHosts/code/phage2taxonomy.py  > phage_host_taxid.txt` and another file `refseq2taxonomy.py` which added the tax id to the list of complete bacterial genomes, and then I made a list of bacteria and taxid:

    cut -f 2,4 /lustre/usr/data/NCBI/RefSeq/bacteria/complete_genome_ids_taxid.txt | perl -pe 's/^.*\|N/N/; s/\.\d+\|//' > bacteria_taxid.txt


I trimmed out any phages that we can't match at the species level:

    python2.7 PhageHosts/code/comparePhageToHosts.py

Then I combined those into a single file for all tax ids

    cat phage_taxid.txt bacteria_taxid.txt > data/all_host_taxid.txt

And add the taxonomy to those files:

    phage_host_add_taxonomy.py all_host_taxid.txt > data/all_host_taxid_taxonomy.txt

*Note* that in this process I deleted two phages whose hosts were not really known (NC_000935) APSE-1 whose host was Endosymbiont and Zamilon virophage (NC_022990) whose host is Mont1 megavirus)

Get the phage coding sequences:

    for i in $(cut -f 1 phage_with_host.tsv); do grep -A 1 $i 2014_07_21/refseq/viral.1.cds.fna; done > data/phage_with_host.cds.fna

Translate the open reading frames:

    perl PhageHosts/code/translate.pl phage_with_host.cds.fna > data/phage_with_host.cds.faa

All bacterial sequences were downloaded from refseq :

    cd NCBI/RefSeq
    ncftpget ftp://ftp.ncbi.nih.gov/refseq/release/bacteria/bacteria.*.fna.gz
    gunzip them and then make a single bacterial database for blastn searches and other stuff
    cat bacteria.*fna > bacteria.genomic.fna


I also downloaded the gbff files and orf files from refseq, and then used those to create a tbl file with both protein and CDS sequences.

    qsub -cwd -o sge/ -e sge/ -S /bin/bash -t 1-152:1 ./get_prots.sh

where get_prots.sh has:

    F=$(head -n $SGE_TASK_ID protein_list | tail -n 1)
    UF=$(echo $F | sed -e 's/.gz//')
    gunzip $F
    perl PhageHost/genbank2flatfile.pl $UF
    gzip $UF

There are a few genomes where some of the orfs are missing for some reason, and so I checked the presence of ORFs with

    python PhageHost/check_dna.py  . | sort | uniq -c  > missed-orfs.txt

Since there are only a few genomes missing more than 1 or 2 ORFs I decided to ignore those and create fasta files of the DNA and proteins.

    python PhageHost/tbl2protdna.py .

This creates the two files, `refseq_proteins.faa` (with proteins) and `refseq_orfs.faa` (with DNA). 

1. Similarity to known proteins (blastx)
========================================

###Part one: against the NR database:
blast complete phage genomes against all bacterial proteins in *nr*, and then
use `gi2tax` to get the taxonomy id of the top hits. Check those against the
expected hosts.

The blast is a standard blast using blast/nr as the database (i.e. all proteins).

To add the tax id, we make a simple script:

	#!/bin/bash
	PYTHONPATH=$PYTHONPATH:$HOME/bioinformatics/Modules
	python PhageHosts/code/blast2taxid.py  phage_with_host.fna.$SGE_TASK_ID.blastx prot

and then run it for all the blast output files:

    qsub -cwd -S /bin/bash -t 1-58:1 ./b2tid.sh 

concatenate all the output files:

    cat phage.blastx/*taxid > phage.host.blastx.taxid

Now we have to convert these to taxids and score the hits:

	for i in all equal best; 
		do echo $i; 
		python PhageHosts/code/blast_hits.py phage.host.blastx.taxid $i ${i}_hits.txt; 
		python2.7 PhageHosts/code/scoreTaxId.py ${i}_hits.txt > ${i}_hits.score;
	done

###Part two: against just the complete genomes:
Start by making a database of just the complete genomes protein sequences

    PhageHost/refseq2complete.py $HOME/phage/host_analysis/bacteria_taxid.txt  refseq_proteins.faa complete_genome_proteins.faa

and then blast those:

    PhageHost/split_blast_queries_edwards_blastplus.pl -f ../../phage_with_host.fna -n 100 -p blastx -d phage.blastx -db /lustre/usr/data/NCBI/RefSeq/bacteria/complete_genome_proteins.faa -evalue 0.01
    cat phage.blastx/*blastx > phage.complete.blastx

NOTE: There are mutliple proteins with the same ID that come from different genomes. How to handle this? Either record a list of those genomes or repeat them multiple times.
To remove those, I just print out unique lines in the blast output file (I had already run the blast):

    perl -ne 'print unless ($s{$_}); $s{$_}=1' phage.complete.blastx > phage.complete.unique.blastx

and then convert so we just have NC identifiers:

    python PhageHosts/code/blastx2genome.py phage.complete.unique.blastx phage.genomes.blastx

Now we just count the sequences with the most number of hits and score those hits

    python PhageHosts/code/mostBlastHits.py phage.genomes.blastx > most.tax
    python2.7 PhageHosts/code/scoreTaxId.py most.tax > score.tax


1b.  Similarity to complete genomes (blastp)
============================================

Use the protein file created above (see `phage_with_host.cds.faa`) and the refseq sequences

    PhageHost/split_blast_queries_edwards_blastplus.pl -f ../../phage_with_host.fna -n 600 -p blastp -d phage.blastp -db /lustre/usr/data/NCBI/RefSeq/bacteria/complete_genome_proteins.faa
    cat phage.blastp/*blastp > phage.complete.blastp

We can then use the blast code from above to summarize these hits and convert them to a score.


2. Similarity to complete genomes (blastn)
==========================================

Get the NC_ ids from the complete bacteria, above:

    grep \> * > ids.txt
    perl -i -npe 's/\:/\t/;s/^(.*)\|\s+/$1\|\t/' ids.txt

Then, I trimmed these down to complete genomes:

    egrep 'complete genome|complete sequence' ids.txt | grep -v plasmid | grep -v 'whole genome shotgun sequence' | grep -v NR_ > complete_genome_ids.txt

We use this set of genomes for the analysis.


for direct matches make a blast db: `makeblastdb -in bacteria.genomic.fna -dbtype nucl` and then blast the complete phages

NOTE: I change the default values here to make them the similar as NCBI blast for a whole genomes, however that usually uses a word size of 28. I cut the word size to 5 so we get some shorter matches

    PhageHost/split_blast_queries_edwards_blastplus.pl -f phage_with_host.fna -n 100 -db /lustre/usr/data/NCBI/RefSeq/bacteria/complete_genomes.fna -d phage.blastn -p blastn -word_size 5 -evalue 10 -num_descriptions 100 -reward 1 -penalty -2 -gapopen 0 -gapextend 0 -outfmt '6 std qlen slen'

There were a few broken blast searches (not sure why) and so a few were missed:

    python2.7 PhageHosts/code/missing_phage_blastn.py phage_host.blastn > missed.phage

and then I cheated to re run these blasts:

    mkdir missed
    for i in $(cat missed.phage | sed -e 's/MISSED //'); do cp ../phage_with_host.fna.files_NC/$i.fna missed/$i; done
    cd missed
    for i in *; do PhageHost/split_blast_queries_edwards_blastplus.pl -f $i -n 1 -db /lustre/usr/data/NCBI/RefSeq/bacteria/complete_genomes.fna -d phage.blastn -p blastn -word_size 5 -evalue  10 -num_descriptions 100 -reward 1 -penalty -2 -gapopen 0 -gapextend 0 -outfmt '6 std qlen slen'; done
    cat phage.blastn/*blastn > missed.blastn

now join those results to the other results:

    cat missed/missed.blastn >> phage_host.blastn

and finally figure out the best hit for each phage using hits.sh which has:

    export PYTHONPATH=:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/
    python2.7 PhageHosts/code/parse_blastn.py phage_host.blastn > phage.hits

and which we submit to the cluster

    qsub -cwd -q important hits.sh

Then score as before:

    python PhageHosts/code/NC2taxid.py phage.hits > phage.taxid
    python2.7 PhageHosts/code/scoreTaxId.py phage.taxid > blastn.score

BLASTN TABULATE:
For the table we just have the total length of sequences that match divided by the length of the phage genome using tabulate.sh:

    export PYTHONPATH=:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/
    python2.7 PhageHosts/code/tabulate_blastn.py phage_host.blastn > blastn.table.tsv

and run ont the cluster:

    qsub -cwd -q important tabulate.sh




3. Exact matches
======================================

I started by finding all 15mer hits between the bacteria and the phages, and then combining those identical hits into longer stretches of similarity.
Basically this two step algorithm finds all identical stretches longer than 15bp between the phage and the bacteria. Then we assume that the integration
site is the longest match for each phage.  We could also score all the longest matches to see what the difference is.

To find all kmers:

    perl PhageHost/find_exact_matches.pl ../phage_with_host.fna '/lustre/usr/data/NCBI/RefSeq/bacteria/complete_genomes.fna' > phage.15mers.bacteria.txt

and then to sort and join the matches we use

    perl PhageHosts/code/sort_and_join_exact_matches.pl phage.15mers.bacteria.txt > phage.kmers.bacteria.txt

to find the longest match per phage:

    perl PhageHosts/code/longest_exact_match.pl phage.kmers.bacteria.txt > phage_best_hits.txt 

Next we convert this to a table of ids:

    perl PhageHosts/code/longest_exact2tbl.pl  > phage_best_hits_NCIDS.txt

scoring; as before we convert to a list of taxa and score the matches:

    python PhageHosts/code/NC2taxid.py phage_best_hits_NCIDS.txt  > phage_best_hits_taxa.txt
    python2.7 PhageHosts/code/scoreTaxId.py phage_best_hits_taxa.txt > score.txt

4. CRISPR Sequences
===================

The spacer database was downloaded from the [CRISPRs WebServer](http://crispr.u-psud.fr/crispr/BLAST/Spacer/Spacerdatabase) and the phage genomes were blasted against that:

    PhageHost/split_blast_queries_edwards_blastplus.pl -f phage_with_host.fna -n 200 -d crispr.blastn -db CRISPR/crispr.u-psud.fr/Spacerdatabase.fna -evalue 10 -outfmt '6 std qlen slen' -p blastn

Use score_blast to convert the blast output to a list of hits:

    python PhageHosts/code/score_blast.py crispr.blastn.rob.blastn best > rob.best.hits

and then crispr_blast2tax.py to convert that output to a list of tax ids and score them

    python PhageHosts/code/crispr_blast2tax.py rob.best.hits > rob.taxid
    python2.7 PhageHosts/code/scoreTaxId.py rob.taxid  > rob.score

Bas decided to make his own crispr database and redo the analysis. The CRISPR database was made using [PILERCR](http://www.drive5.com/pilercr/) He provided a series of tsv files that are NC_ ids one per phage, so I just need to score them:

    for i in *tsv; do
	    t=$(echo $i | sed -e 's/tsv/taxid.txt/'); 
	    s=$(echo $i | sed -e 's/tsv/score.txt/'); 
	    echo -e "$i\t$t\t$s";
	    python PhageHosts/code/NC2taxid.py $i > $t;
	    python2.7 PhageHosts/code/scoreTaxId.py $t > $s;
    done


Use score_blast to convert the blast output to a list of hits:

    python PhageHosts/code/score_blast.py crispr.blastn.bas.blastn best > bas.best.hits

and then crispr_blast2tax.py to convert that output to a list of tax ids:

    python PhageHosts/code/crispr_blast2tax.py bas.best.hits > bas.taxid

and then score that:

    python2.7 PhageHosts/code/scoreTaxId.py bas.taxid  > bas.score


5. GC Content of Coding Regions
===============================

Similar to codon usage below, we just calculate the G+C/G+C+A+T content of the coding regions. This gives us a simple one dimensional matrix that we can use.

To count the GC contents:

    python PhageHosts/code/cds_gc.py ../phage_with_host.cds.fna  > phage.gc.tsv

bacteria.sh:

    DIR=/lustre/usr/data/NCBI/RefSeq/bacteria
    python PhageHost/cds_gc.py $DIR/bacteria.$SGE_TASK_ID.orfs.fna.gz > bacteria/bacteria.$SGE_TASK_ID.gc.tsv

and then

    qsub -cwd -o sge -e sge -t 1-152:1 -S /bin/bash ./bacteria.sh


Then pull out the complete bacteria from this list:

    python PhageHosts/code/complete_bacteria.py  > complete_bacteria_gc.tsv

Now we have two files: complete_bacteria_gc.tsv  phage.gc.tsv and we can just get the closest organisms

    python PhageHosts/code/cds_distance.py phage.gc.tsv complete_bacteria_gc.tsv  > closest_genomes.ids

Convert to taxa  and score as before

    python PhageHosts/code/NC2taxid.py closest_genomes.ids > closest_genomes.taxa
    python PhageHosts/code/scoreTaxId.py closest_genomes.taxa > score.txt


6. Codon usage
==============

for codon counts:
use `PhageHosts/code/codon_usage.py` to count the codons and print out tables. Then we just need to calculate the nearest neighbor by euclidean distance or some other distance measure

    python PhageHosts/code/codon_usage.py ../phage_with_host.cds.fna  > phage_codon_counts.tsv

and then count the distance using:

    python PhageHost/codon_distance.py phage_codon_counts.tsv bacteria_codon_counts.tsv  > phage_bacteria_predictions.out

convert that output to taxids and then score:
    python PhageHosts/code/NC2taxid.py phage_bacteria_predictions.out > phage_bacteria_predictions.taxid
    python2.7 PhageHosts/code/scoreTaxId.py phage_bacteria_predictions.taxid > score.txt

7. Kmer profiles
================

for kmer-counts the longest phage sequence is 358,663 bp, so we'll use a hash size of 400000

Initially we start with this script to list all the 3-mers. 

    for f in $(ls ../phage_with_host.fna.files_NC/*);
	    do n=$(echo $f | sed -e 's/\.\.\/phage_with_host.fna.files_NC\///; s/fna$/tsv/');
	    echo $f $n;
	    jellyfish count -s 400000 -t 32 -C -m 3 -o 3mers.txt $f;
	    jellyfish dump -ct 3mers.txt_0 > 3mers/$n;
    done

and then we modify it to count all k-mers between 3 and 20. That is in
count_kmers.sh and runs on a node of the cluster in a few minutes.


    #!/bin/bash
    ## Count all kmers between 3 and 20 and put the output in a single directory
    export PATH=$PATH:$HOME/bin
    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/lib
    
    ## first make all the directories
    for k in $(seq 3 20); do mkdir ${k}mers; done
    ## loop through each file
    for f in $(ls ../phage_with_host.fna.files/*); do 
        n=$(echo $f | sed -e 's/\.\.\/phage_with_host.fna.files\///; s/fasta$/tsv/'); 
        #echo $f $n; 
        for k in $(seq 3 20); do 
                jellyfish count -s 400000 -t 32 -C -m $k -o mers.txt $f;  
                if [ -e mers.txt_1 ]; then
                        jellyfish merge -o output.jf mers.txt_*
                else
                        mv mers.txt_0 output.jf
                fi
                jellyfish dump -ct output.jf  > ${k}mers/$n; 
                rm -f mers.txt_* output.jf
        done
    done


We then combine those outputs to a single file using combineKmerCounts.py.
This takes a while so we run it on the cluster by making combineKmerCounts.sh

    #!/bin/bash
    python2.7 $HOME/bioinformatics/phage_host/combineKmerCounts.py phage_kmer_counts/${SGE_TASK_ID}mers all_phages/${SGE_TASK_ID}mers.txt 0

and then running on the cluster:

    qsub -q important -cwd -t 3-20:1 ./combineKmerCounts.sh

That makes a single .txt file for each kmer size.

To count all bacterial kmers we need to extract the fasta sequences to
individual files and run jellyfish on all of those. The combination of bash
and perl does that. Make count_bacterial_kmers.sh:

	#!/bin/bash
	## Count all kmers between 3 and 20 and put the output in a single directory
	export PATH=$PATH:$HOME/bin
	export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/lib
	export PERL5LIB=$PERL5LIB:$HOME/bioinformatics/Modules
	
	BACTGZ=$(head -n $SGE_TASK_ID bacterial_genomes.txt | tail -n 1)
	BACT=$(echo $BACTGZ | sed -e 's/.gz//')
	
	if [ ! -e kmers ]; then mkdir kmers/; fi
	if [ ! -e sequences ]; then mkdir sequences/; fi
	
	echo "$SGE_TASK_ID : Processing $BACT"
	mkdir sequences/$SGE_TASK_ID.sequences
	./split.pl $BACTGZ sequences/$SGE_TASK_ID.sequences
	
	mkdir kmers/$SGE_TASK_ID.kmers
	for FILE in $(ls sequences/$SGE_TASK_ID.sequences/); do
	        for k in $(seq 3 10); do 
	                jellyfish count -s 400000 -t 32 -C -m $k -o kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer sequences/$SGE_TASK_ID.sequences/$FILE
	                if [ -e kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer_1 ]; then
	                        jellyfish merge -o kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer.output.jf kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer_*
	                else
	                        mv kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer_0 kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer.output.jf
	                fi
	                jellyfish dump -ct kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer.output.jf > kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer.tsv
	                rm -f kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer_*  kmers/$SGE_TASK_ID.kmers/$FILE.${k}kmer.output.jf 
	        done
	done

and we run it on the cluster:

    qsub -S /bin/bash -o sge_out -e sge_error -cwd -t 1-165:1 ./count_bacterial_kmers.sh

This generates a single tsv file for every genome that we are going to use in
the analysis.

Move kmer counts into individual directories for 3 .. 20 mers and remove empty
directories:

    for i in $(seq 1 20); do mkdir -p kmers/$i; done
    for i in $(seq 1 20); do mv complete_genome_kmers/*.${i}kmer.tsv kmers/$i/; done
    rmdir complete_genome_kmers/
    rmdir kmers/*

Finally, I just make sure we only include genomes that we are interested in
for our analysis.

    for i in $(ls all_phages); do echo $i; python trim.py all_phages/$i all_phages_trimmed/$i; done
    for i in $(seq 3 9); do python trim.py bacterial_genomes/kmers/$i.kmers bacterial_kmers/$i.kmers; done

The results are in bacterial_kmers and phage_kmers


8. Cooccurrence analysis
=========================

We use FOCUS (Silva et al from Rob's lab) to predict the hosts in the metagenomes, but we need to make unique FOCUS databases first:

    cp -r ~/bin/focus/bin focus_bacteria
    cd focus_bacteria/db/
    mv k6 k6_old
    mv k7 k7_old
    mv k8 k8_old
    head -n 1 k6_old > k6
    head -n 1 k7_old > k7
    head -n 1 k8_old > k8
    cd ../..
    grep \> NCBI/RefSeq/bacteria/complete_genomes.fna.files.NC/*  > complete_genomes_files.txt
    perl PhageHosts/code/focus_grep2list.pl < complete_genomes_files.txt  > complete_genomes_files.focus.txt

then make a shell script to run focus. Don't forget to set LD_LIBRARY and PYTHONPATH:

    env | grep LD_L > fdb.sh
    env | grep PYTHONP >> fdb.sh
    echo "python2.7 focus_bacteria/focus.py -d complete_genomes_files.focus.txt" >> fdb.sh

then run that on the queue

    chmod +x fdb.sh
    qsub -cwd -q important ./fdb.sh

Now we can run all the bacterial genomes using focus. This script makes a directory for each job, cd's there, copies the data, runs focus, and then cleans up.
Note also that I put a new line "Input: ...." at the beginning of the output so I know which job was with which sample


This is focus.sh
    #!/bin/bash
    export LD_LIBRARY_PATH=:$HOME/lib
    export PYTHONPATH=:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/:/usr/local/opencv/lib/ python2.6/site-packages/:$HOME/bioinformatics/Modules/
    export PATH=$PATH:$HOME/bin
    GZFILE=$(head -n  $SGE_TASK_ID mg-rast.txt | tail -n 1)
    mkdir /scratch/$SGE_TASK_ID
    cd /scratch/$SGE_TASK_ID
    gunzip -c $GZFILE > $SGE_TASK_ID.fasta
    echo "Input: " $GZFILE  > cooccurence/focus_NC/$SGE_TASK_ID.focus
    python2.7 focus.py -m 0 -q $SGE_TASK_ID.fasta >> cooccurence/focus_NC/$SGE_TASK_ID.focus
    cd $HOME/phage/host_analysis/cooccurence
    rm -rf /scratch/$SGE_TASK_ID

Note there are 3025 files in mg-rast.txt, the list of all metagenomes we have, so we use head and tail to get a single one based on the SGE_TASK_ID. This is why we use SGE.

    qsub  -cwd -S /bin/bash -t 1-3025:1 -o sge -e sge ./focus.sh 

To convert these to a matrix, use 

    python PhageHosts/code/focus_parse.py focus_NC2 > focus_bacteria_predictions.tsv

note: I check the dimensions of this file with:

    perl -ne'@a=split /\t/; print "$#a\n"' focus_bacteria_predictions.tsv | sort | uniq -c

To print a few summary statistics about the results:

    python PhageHosts/code/focus_summary.py focus_bacteria_predictions.tsv


## Comparing to BLAST
Kate has already run megablast for all phages against all metagenomes. I start by checking that she included all phage genomes we need
Her output is in phage_hits/out_blastn

    mkdir blast
    cd blast
    cut -f 1 ../../phage_with_host.tsv  > phage_ids.txt
    mkdir blast_results
    python PhageHosts/code/extract_phage_mg_blast.py phage_hits/out_blastn phage_ids.txt blast_results 2> blast_check.err

Note: I did this a couple of times as we added genomes that we had missed.
Finally: `python PhageHosts/code/extract_phage_mg_blast.py interim_blast_results/ phage_ids.txt blast_results/ 2> missed`


Once the blast is complete we can count the hits:

    python2.7 PhageHosts/code/count_phage_mg_hits.py 2> err

This makes two tables with an additional column, the taxonomy of the phage, which Karoline Faust wanted me to include.
The tables are `raw_phage_mg_counts.tsv` and `normalized_phage_mg_counts.tsv`. The first is just the number of times that phage is seen in each metagenome, and the second is the number divided by the number of reads in the metagenome. It is the normalized number we need (longer metagenomes have more hits, after all). 

Calculating the co-occurrence correlation
-----------------------------------------

We have two files:
`normalized_phage_mg_counts.tsv` and `focus_bacteria_predictions.tsv` that we need to compare. Note that `focus_bacteria_predictions.tsv` is in percentages, and phage_hits is the fraction of the metagenome that matches that phage

Calculate the pearson correlation between genomes and phages:

use pearson.sh to create the occurence. This contains:

    export PYTHONPATH=:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/
    python PhageHost/cooccurrence_pearson.py focus_bacteria_predictions.tsv normalized_phage_mg_counts.tsv

and then convert to refseq ids, taxonomy ids, and score as before

    python PhageHost/scoreDistances.py pearson_correlations.tsv max > pearson.ncids 
    python PhageHosts/code/NC2taxid.py pearson.ncids > pearson.tids
    python2.7 PhageHosts/code/scoreTaxId.py pearson.tids > pearson.score


We also have the code all_coorccurence.sh to calculate all the different distance measures:
export PYTHONPATH=:/usr/local/opencv/lib/python2.6/site-packages/:$HOME/bioinformatics/Modules/
python PhageHost/cooccurrence_all.py focus_bacteria_predictions.tsv normalized_phage_mg_counts.tsv

Score all the correlations for the different distance measures

    for i in euclidean braycurtis cityblock; do python PhageHost/scoreDistances.py ${i}_correlations.tsv min > ${i}_correlations.ncids; python PhageHosts/code/NC2taxid.py ${i}_correlations.ncids > ${i}_correlations.tids; python2.7 PhageHosts/code/scoreTaxId.py ${i}_correlations.tids > ${i}_correlations.score; done



Mutual Information
------------------

Based on an idea by Peter Salamon from SDSU:

You need a threshold of present in the metagenome to turn your current variables into 0,1 variables that signal presence of the entity in the metagenome. As a start, you could threshold on mean value for that entity (phage or bacterium). Then you need the fraction of metagenomes in which the phage and the bacterium co-occur, the fraction with one not the other and the fraction with the other and not the one and the fraction with neither. This gives the joint distribution of the four cases. You also need the marginal distributions of fraction in which the phage occurs and the fraction in which the bacterium occurs. Then simply calculate the Kullback entropy of the joint distribution relative to the product distribution.

    python2.7 PhageHost/mutual_information.py normalized_phage_mg_counts_scaled.tsv focus_bacteria_predictions.tsv 0 > mi.mean.ncids
    python PhageHosts/code/NC2taxid.py mi.mean.ncids > mi.mean.tids
    python2.7 PhageHosts/code/scoreTaxId.py mi.mean.tids > mi.mean.score




