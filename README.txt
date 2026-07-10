# Repurposing Splicing Modulator Compounds (SMCs) from RNA-Seq Data    
    
===========================================================================
INTRODUCTION
This README file demonstrates how to regenerate data and figures described in the paper "A deep learning approach to identify new gene targets of a novel therapeutic for human splicing disorders".


===========================================================================
DEPENDENCIES
1. R (>=3.5).
2. Python (version = 2.7).
3. Samtools (>=1.4).
4. Basset (https://github.com/davek44/Basset).
5. SpliceAI (https://github.com/Illumina/SpliceAI).

Note: 
1. This package contains data processing steps that are NOT suitable to run on a laptop or desktop. Please run this package on a computational workstation or node!   
2. This package has only been tested on Linux platforms.
3. Other R and Python packages might be required by scripts in this package, depending on users' system.
4. To install any missing R package, please use "install.packages('PACKAGE_NAME')".
5. To install any missing Python package, please use either "conda install PACKAGE_NAME" for anaconda system or "pip install PACKAGE_NAME".


===========================================================================
DESCRIPTION
This package contains eight folders:
1. Input_data: contains original input file upon which the entire set of data and figures of the paper can be reproduced. There are six files come with this package:
   a. allSpliceIDs.txt: the entire list of human exon triplets according to Ensembl GRCh37 release 75.
   b. alternative_exon_tripletIDs.txt: a list of human exon triples whose middle exon is skippable in WT according to Ensembl GRCh37 release 75 (i.e. there is transcript with the middle exon skipped but with the upstream and downstream exons retained).
   c. SEMatrix_15477onWT.txt: read counts supporting each human exon triplet, summaried from RNASeq data of this paper.
   d. grch37.txt: Start and end position for each human gene according to Ensembl GRCh37 release 75. This is required by SpliceAI.
   e. clinvar_20190325_SNV_INDEL1000_mainChr_headerLength.vcf.gz: a processed ClinVar annotation (derived from ClinVar version 20190325). Details below.
   f. var_citations_20190325.txt: original publication ID (e.g. PubMed ID) for each ClinVar mutation (version 20190325).
   
2. Output_data: Originally empty. Data output of each analysis reproducing step will be saved here.
3. Output_figures: Originally empty. Reproduced figures will be saved here.
4. Step01_Differential_splicing_analysis: contains scripts to reproduced differential splicing analysis.
5. Step02_Build_CNN_model: contains scripts and hyperparameters to rebuild the CNN model in the paper.
6. Step03_Find_ClinVar_splicing_mutations: contains scripts to identify ClinVar pathogenic mutations that influence splicing.
7. Step04_Identify_new_targets: contains scripts to identify potential therapeutic targets for BPN15477.
8. Step05_Make_figures: contains script to reproduce figures in the paper. 

Note:
1. To regenerate all results for the paper, the scripts from Step01 to Step05 have to be run sequentially.
2. To reproduce a result of a certain step, please run all the steps prior to it.
3. Please keep the file structure of the package, as input and output paths are coded relatively in many of the scripts.
4. To run any R script in this package, please make sure the working directory is set to where the script sits.


===========================================================================
DETAILS
Before start, let's define some global parameters and download some refernece files:
1. $MY_FOLDER: the full path to this package
2. $BASSET_FOLDER: the full path to the Basset package.
3. Download genomic FASTA file for GRCh37 release 75 from ftp://ftp.ensembl.org/pub/release-75/fasta/homo_sapiens/dna/Homo_sapiens.GRCh37.75.dna.primary_assembly.fa.gz to "Input_data" folder unzip it.
4. Rename the unzipped genomic FASTA file to "GRCh37.fa" and index it by the following commnad:
   samtools faidx GRCh37.fa
 
Step00. Preprocessing
This step is not included in this package, due to the file size limitation. Anyway, the regeneration process is very straightforward.
Raw data from RNASeq reads have been prepocessed in step. 
First, each RNASeq library was aligned aginast Ensembl GRCh37 (release 75) using STAR (version 2.5.2a) using the following parameters:
	--runThreadN 8 
	--twopassMode None 
	--outFilterMultimapNmax 1 
	--outFilterMismatchNoverLmax 0.05
	--outSAMtype BAM Unsorted 
	--outReadsUnmapped Fastx  
	--alignEndsType Local
Second, numbers of spliced reads supporting R1, R2, R3 of each annoated exon triplet (Fig. 2a in the paper) were extracted from SJ.tab.out file generated in each STAR alignment folder. 
These numbers for each sample were then merged into a single matrix, leading to the file "Input_data/SEMatrix_15477onWT.txt" in this package.


Step01. Differential Splicing Analysis
This step identifies differntially spliced middle exon of an exon triplet. 
It also finds negative training set where the splicing change of the middle exon is miminal.
To regenerate the result, please run through the R script "fisherBased_psiTest.R" in "Step01_Differential_splicing_analysis".
It will generates a bundle of output files in the "Output_data" folder of this package.
The key output files are:
1. up01tripletNames_pos_fdr01.txt: for those exon triplets whose PSI upregulation of the middle exon no less than 0.1 and FDR less than 0.1, namely "Inclusion" in the papper.
2. dn01tripletNames_pos_fdr01.txt: for those exons triplets whose PSI downregulation of the middle exon no less than 0.1 and FDR less than 0.1, namely "Exclusion" in the papper.
3. dn01tripletNames_pos_fdr01.txt: for those exons triplets with minimal PSI change, namely "Unchanged" in the papper.


Step02. Build CNN Model
The following aims are achieved in this step:
 1. Build one-hot-coded matrix for training data
   a. Go to "Step02_Build_CNN_model" folder.
   b. Open "makeFiles4Basset.py" and change Line 9 (BASSET_FOLDER = "") to the real path of $Basset_FOLDER, and then run the following:
      cd $MY_FOLDER/Step02_Build_CNN_model
      python makeFiles4Basset.py
	  
      This creates file "Output_data/3C_learn.h5", which is the training data for the CNN model	  

 2. Train the CNN model
   The CNN model is trained using the hyperparameters defined in "Step02_Build_CNN_model/param38.txt". To train the model, please run the following:
   cd $BASSET_FOLDER/src/
   ./basset_train.lua\
    -job $MY_FOLDER/Step02_Build_CNN_model/param38.txt\
    -save $MY_FOLDER/Output_data/3C_cnn_param38\
    -result $MY_FOLDER/Output_data/3C_cnn_loss_param38\
    -rand 122\
    $MY_FOLDER/Output_data/3C_learn.h5
	
	The trained model is saved at "Output_data/3C_cnn_param38_best.th"
 
 3. Identify motif learned by the CNN model
   Copy all files in the "Step02_Build_CNN_model/Basset_core" to "$BASSET_FOLDER/src". This includes:
   a. basset_motif_convolution_dadi.py
   b. basset_motifs_dadi.py
   c. basset_sad_dadi.py
   d. basset_sat_vcf_dadi.py
   e. bvcf_dadi.py
   For a, b, c, and d, please open each file and change Line 19 (os.environ['BASSETDIR']="") to the real path of $Basset_FOLDER.
   These customized scripts expand Basset core functions to generate the data in the paper.
   Also copy the file "Step02_Build_CNN_model/rbp_consensus_human.meme" to "$BASSET_FOLDER/data/motifs", which provides Basset with extra reference of common motifs for RNA binding proteins. 
   Then run the following command to extract motif information learnt by the CNN model:
   python basset_motifs_dadi.py -s 131\
    -o $MY_FOLDER/Output_data/3C_motifs_param38\
    $MY_FOLDER/Output_data/3C_cnn_param38_best.th\
    $MY_FOLDER/Output_data/3C_learn.h5
	
	This generates folder "Output_data/3C_motifs_param38". The folder contains top 50 motifs learnt by the CNN model, named as "filter0" to "filter49".
	Note: 
	1. Not all of 50 motifs contribute to the treatment response. We forced the model to find 50. 
	2. Read the following session about how we determine if a motif has a contribution to the treatment.
	
 4. Contribution of motif to treatment response
  python basset_motifs_infl.py -s 131 -i\
   -o $MY_FOLDER/Output_data/3C_motifs_infl_param38\
   $MY_FOLDER/Output_data/3C_cnn_param38_best.th\
   $MY_FOLDER/Output_data/3C_learn.h5
   
   This generates folder "Output_data/3C_motifs_infl_param38". Inside the folder, the file "table.txt" records the contribution of each motifs to treamtment response. 
   The five columns of this file represent motif name, entropy, average contribution, contribution standard deviation, information content of the motif and the annotation of the motif in database, respectively.
   We considered motifs with average contribution no less than 0.01 as "real" motifs for the treatment, which ended up 39 out of 50 motifs.

 5. Influence of each position for treatment response
  python basset_motif_convolution_dadi.py \
   -o $MY_FOLDER/Output_data/3C_positional_weight_param38\
   $MY_FOLDER/Output_data/3C_cnn_param38_best.th\
   $MY_FOLDER/Output_data/3C_learn.h5
   
   This generates "Output_data/3C_positional_weight_param38/positional_infl_entropy_heatmap.pdf". It contains four panels from left to right.
   For each panel, columns represent positions along a 400bp region of UI1, I1X, XI2 and I2X (Fig. 2a) while rows represent motif 18, 25, 49, 9, 10, 21, 29, 47, 1, 22, 27 and 37 (top to bottom), respectively (Fig. 3b).
 
 6. Determine the cutoff for inclusion, exclusion and unchanged drug response
 For each exon triplet, the drug responses as inclusion, exclusion and unchanged are predicted independently. 
 To determine which category can be used to represent the final drug response, we first have to normalize the prediction score for each category as the training size for each category is imbalanced.
 We first examined the performance of our model on prediction of ground truth observed from RNASeq, namely the test set of data that was not used during the model building, using the following command:
  python basset_predict.py\
    $MY_FOLDER/Output_data/3C_cnn_param38_best.th\
    $MY_FOLDER/Output_data/3C_learn.h5\
    $MY_FOLDER/Output_data/3C.test0.fa.param38.prob
	
 Then we investigated the point on each of the AUC curve represents 95% prediction specificity of each category.
 This set of cutoff was used to normalize the prediction results and drew conclusions on drug response for each exon triplet.
 The R script "$MY_FOLDER/Step02_Build_CNN_model/probThreshold.R" can be used to find the cutoff for each category prediction.

Step03. Find ClinVar Splicing Mutations
Please first uncompressed "Input_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength.vcf.gz". This is a processed ClinVar VCF file, deriving from version 20190325.
It has been filtered from the original file to only retain SNVs and INDELs less than 1000bp and on main chromosomes (e.g. 1-22, X and Y). The chromosome length is also added to the header of the file, to be compatible with SpliceAI.
To predict which ClinVar mutation might influence splicing, please run the following command:
  spliceai -I $MY_FOLDER/Input_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength.vcf -O $MY_FOLDER/Output_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength_spliceAIpredicted.vcf -R $MY_FOLDER/Input_data/grch37.fa -A grch37

WARNING: 
a. DO NOT run the above command on laptop or a normal desktop!!! The above command might take 2-4 weeks to complete and always consumes 16 cores and 64GB memory durin the process. Run it on a computational cluster instead!
b. To accelerate the process, please split clinvar_20190325_SNV_INDEL1000_mainChr_headerLength.vcf into several subsets and run the above command parrellely on each of them. Then merge all the predicted output at the end.

The output file "$MY_FOLDER/Output_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength_spliceAIpredicted.vcf" contains prediction score representing the strongest influence of each mutation on the splicing of the nearby (within 10kb) splice sites.
Here, we further filtered this predicted result to find mutations meeting all of the following criteria:
1. Annotated as pathogenic or likely pathogenic mutations in ClinVar
2. With a SpliceAI score no less than 0.2, according to the original SpliceAI paper
3. The scored site (the GT/AG site whose splicing is predicted to be influenced by the mutation) is an annotated splice site according to Ensembl GRCh37 release 75.

To apply the above filter on the SpliceAI predicted result, please run the following command:
  cd $MY_FOLDER/Step03_Find_ClinVar_splicing_mutations
  python filterSpliceAIpredictedVCF4BassetSAD_param38.py


Step04. Identify Potential Therapeutic for BPN15477
In Step03, it generates "$MY_FOLDER/Output_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength_spliceAIpredicted_ds02_pathogenic_junctions_mEX_4BassetSad.vcf", which is ready for treatment response prediction using our trained CNN model in Step02.
To apply the CNN prediction, please run the following command:
  cd $BASSET_FOLDER/src/
  python basset_sad_dadi.py -l 400 -i -s\
   -o $MY_FOLDER/Output_data\
   $MY_FOLDER/Output_data/3C_cnn_param38_best.th\
   $MY_FOLDER/Output_data/clinvar_20190325_SNV_INDEL1000_mainChr_headerLength_spliceAIpredicted_ds02_pathogenic_junctions_mEX_4BassetSad.vcf
   
This generates a raw output "$MY_FOLDER/Output_data/sad_table.txt", which contains predicted treatment response for each mutation passing the filter in Step03. 
We futher filtered the raw result based on the strength and the direction of the predicted drug responses by the following command:
  cd $MY_FOLDER/Step04_Identify_new_targets
  python filterFinalRescue.py
  
This creates two files, "$MY_FOLDER/Output_data/sad_table_inclusion.txt" and "$MY_FOLDER/Output_data/sad_table_exclusion.txt", representing potential rescue by inclusion and exclusion effect of the treatment, respectively.
We merged the two files by the command:
  cat <(cat $MY_FOLDER/Output_data/sad_table_inclusion.txt) <(tail -n +2 $MY_FOLDER/Output_data/sad_table_exclusion.txt) > $MY_FOLDER/Output_data/sad_table_rescue.txt
  
We further added allele frequency to each mutation.
Please first download Gnomad exome and genome variant data from the following address (WARNING: 60G and 460G respectively) to "$MY_FOLDER/Input_data": 
https://storage.googleapis.com/gnomad-public/release/2.1.1/vcf/exomes/gnomad.exomes.r2.1.1.sites.vcf.bgz
https://storage.googleapis.com/gnomad-public/release/2.1.1/vcf/genomes/gnomad.genomes.r2.1.1.sites.vcf.bgz
Then allele frequencies can be integrated by the command:
  cd $MY_FOLDER/Step04_Identify_new_targets
  python rescuedClinVarID_gnomADfreq.py
  
Finally, we got rid of exon triplets whose middle exon is NOT skippable in WT from the result of potential rescue by exclusion effect by the following command:
  python makeSupplemenaryTable_rescueSet_wAltExon.py
  
This generates the final list for all potential therapeutic of BPN15477 at "$MY_FOLDER/Output_data/sad_table_rescue_alleleFreq_4supplementaryTable_wAltExon.txt"


Step05. Make Figures of the paper
Paper figures can be generated by running through the R script at "$MY_FOLDER/Step05_Make_figures/finalFigures.R" 

Note: 
1. Some figures are generated by Step01-Step04.
2. Wet-lab-derived figures are NOT covered by this package.


===========================================================================
CONTACT
For any question, please contact Dadi Gao by email: 
  dgao2 at mgh dot harvard dot edu

  
  
  



  









