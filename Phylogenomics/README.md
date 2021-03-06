# Intro: Phylogenomics pipeline 
This is how I modifed the Spatafora et al. 2016 pipeline, which can be found here: https://github.com/zygolife/Phylogenomics/tree/master/Spatafora_et_al_2016

**NOTE: You will need to install trimAl from http://trimal.cgenomics.org/ as this program is not available on the flux. I installed v1.2.**

The directory structure I work in:
- phylogenomics
    - aln
    - aln_final
    - HMM3
    - jobs
    - messy_pep
        - genbank
        - jgi
        - Maker
    - pep
    - raxml_gene_trees
    - raxml_gene_tree_scripts
    - scripts
        - combinefasaln.pl (from Spatafora et al. 2016)
        - construct_unaln_files.pl (from Sparafora et al. 2016)
        - get_best_hmmtbl.pl (from Spatafora et al. 2016)    
    - search
    - sh_scripts
    - Tree

## Step 1: Cleaning up the sequence names

First, you need to clean up the sequence names in all of your amino acid fasta files. The sequence names should follow the format of >Organism_name|locus. This will require different manipulations depending on where your data came from. Second, this pipeline requires that all of the fasta files have the same wrap length. The easiest way to ensure this is to place the sequence on a single line. I recommend doing this within the `/messy_pep` directory with the raw files in the appropriate sub-directories (i.e., files from Genbank in the `/genbank` directory).

### For Maker files:
   1.  To clean up the sequence names from the horribleness that is Maker by running the following commnand:
```
cat [protein_fasta] | awk '{print $1}' > [organism_name]_renamed_seqs.aa.fasta
```   
   2. To make it single line: 
```
cat [organism_fasta]_renamed_seqs.aa.fasta | awk '/^>/ {printf("\n%s\n",$0);next; } { printf("%s",$0);} END {printf("\n");}' > [organism_name]_single_line.aa.fasta
```
   3. To put it in [organism_name]|[locus] format:
```
cat [organism_name]_single_line.aa.fasta | sed 's/_/|/g' > [organism_name]_final_names.aa.fasta
```
### For Genbank fastas (example scripts can be found in the "processing_genbank_files" folder https://github.com/Michigan-Mycology/Lab-Code-and-Hacks/tree/master/phylogenomics/processing_genbank_files):
  - To get just the acession numbers (which will serve as the "locus" name) which can be done as a for loop in a shell script.
```
cat  [genbank_name].fa | awk '{print $1}' | sed 's/\.1/g' > [genbank_name]_renamed1.aa.fasta
```
  - To put the sequence names in [organism_name]|[locus_name] format. I had to do this for each file individually. If you are clever, I am sure there is a way to write it into a script.
```
cat [genbank_name]_renamed1.aa.fasta | sed 's/>/>sp_name\|/g' > sp_name_penultimate_names.aa.fasta
```
  - To make it all single line, which can be done as a script.
```
cat sp_name_penultimate_names.aa.fasta | awk '/^>/ {printf("\n%s\n",$0);next; } { printf("%s",$0);} END {printf("\n");}' > [organism_name]_final_names.aa.fasta
```
### For JGI files

[under construction]


## Step2: hmmer search
   1. Finally move/copy all of the `*final_names.aa.fasta` files into the `/pep` directory. You will need to populate the `/HMM3` folder with the `*.hmm` files containing the markers you wish to use.  I used the markers from Spatafora et al. 2016, which can be found here https://github.com/zygolife/Phylogenomics/tree/master/Spatafora_et_al_2016/HMM/Roz200. There are other markers available for fungi here:
- Make sure the `markers_3.hmmb` file is in the `/phylogenomics` directory.
- You will need to load the following modules (current versions are preferable).
```
module load hmmer
module load bioperl
```
   2. The next step is to perform a hmmer search for all of the markers in your genomes of interest. Hmmer uses hidden Markov models to find the markers in your genomes of interest. The page can be found here: http://hmmer.org/ I re-wrote the Spatafora et al. script for this, and it is placed in this folder.
  
  ```
  sh hmmsearch_loop.sh
  ```
  
  3. This should produce non-zero .dombtl & .log files for each of your genomes. We only want the best hit, i.e., the best match to the hmmer model, for each genome. We can get that by calling the "get_best_hmmtbl.pl" script written for the Spatafora et al. paper. This script should be placed in your `/scripts` directory. I have included my script for doing so in this folder.

  ```
  sh make_get_best_hits.sh
  ```
  4. This should produce non-zero .best files for each of your genomes. Move (or copy) the .best files into the `search` directory. Currently, we have a list of each marker for each genome, but we want a fasta file of each marker. To do this, we need the `construct_unaln_files.pl` from the Spatafora et al. github. Place it in your `/scripts` directory. My script for calling the `construct_unaln_files.pl` script is in this folder.
  ```
  sh make_unaln.sh
  ```
  5. This should populate the `/aln` directory with non-zero fasta files--one for each marker. Now you will need to align all of the markers. I could not get the script in Spatafora et al.'s github for this to work. So, I created a work around. First, you will need to generate a space-delimited list of the markers in the `/HMM3` directory.
  ```
  ls > LIST
  cat LIST | tr "\n" "\t" | sed 's/\t/ /g' > space_list
  ```
   6. Using vi (or vim or nano), copy the space-delimited list into the `make_hmmalgn.sh` script in this folder. Then run the `make_hmmalgn.sh` script.
  ```
  sh make_hmmalgn.sh
  ```
This will generate .sh files in the `/sh_scripts` directory. There should be one for each `.hmm` file.

7. Now, run them using the `run_sh_scripts.sh` script.
```
sh run_sh_scripts.sh
```
This should create an alignment of each fasta file in the `/aln` directory and save it as a `.msa`.

8. Next, you need to clean up the alignments using a combination of easel (part of hmmer) and trimAl (which you will need to install).
```
#Example scripts for doing this are provided in this folder.
sh esl_reformat1.sh
sh esl_reformat2.sh
sh trimal_1.sh
sh trimal_2.sh
```
You now have clean alignments of all of your markers! 

## Step 3: Concatentation
From here, you can concatenate all of the markers into one large alignment, which I explain in the `how_to_concatenate_algns` sub-folder https://github.com/Michigan-Mycology/Lab-Code-and-Hacks/tree/master/phylogenomics/how_to_concatenate_algns . Or you can infer gene trees. I recommend inferring the gene trees first so that you cna identify potential paralogues and contamination before making the concatenated alignment and infer the final tree. 

  1. Copy all of the `*.msa.trim` files into the `/aln_final` directory. Run the `make_lots_of_raxml_pbs.sh` script after   modifying it appropriately.
 ```
 sh make_lots_of_raxml_pbs.sh
 ``` 
  2. To infer the trees, you will need to load some modules.
    ```
    module load raxml gcc/5.4.0 openmpi/1.10.2/gcc/5.4.0
    ```
    **NOTE: It is best practice to be sure that the scripts will run/raxml is called properly before submitting all of these    pbs scripts to the queue.** 

  3. Once you are satisfied that the jobs will run without error, submit them.
```
sh qsub_loop.sh
```
4. Once all of the gene trees have been inferred, inspect them. 
```
[under construction]
```
5. Once you have used the gene trees to make informed decisions about the markers to keep/toss, correct paralog problems, and remove contamination (see above), it is time to concatenate the markers into one alignment.
*More detailed instructions can be found here https://github.com/Michigan-Mycology/Lab-Code-and-Hacks/tree/master/phylogenomics/how_to_concatenate_algns*
```
sh combine_make_superaln.sh 
```
The perl script is verbose; it may spit out a bunch of lines. Many of the lines will say something about the "search" and "expect" lists are not the same length. This is to be expected; not every marker is present in envery genome. I have tried to direct the stout to a log file. If that is unsucessful, just copy and paste the stout into a text file.

  6. Move/copy the concatenated alignment into the `/Tree` directory.
  7. Modify the `phylogenomics_tree.pbs` script appropriately.
  8. Submit and wait for your beautiful tree to appear!
