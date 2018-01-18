# Brief step-by-step instructions on Qiime data processing
1. Enter your meta data into the the [mapping file using Google docs ](https://docs.google.com/spreadsheets/d/1AIN9XGXPXAuC5phEK36AEsDgUIEfAg_RY-ZbD36IssY/edit?usp=sharing). Download the mapping file as tab-separated value file (~ file > download as > tsv). Suggested name: `map.tsv`
2. Log onto Cirrus and create a working directory in your home directory, e.g `ssc`. **Important:** do not enter your password incorrectly, or you and your colleagues might be locked out.
    ```bash
    ssh username@login.cirrus.ac.uk
    mkdir ~/ssc
    ```
3. Open an new terminal window (GitBash or terminal or Putty) and upload the mapping file to Cirrus using secure copy; e.g.
    ```bash
    scp map.tsv username@login.cirrus.ac.uk:~/ssc/

    ```
4. On the Cirrus terminal and load the miniconda/python2 module
    ```bash
    module load miniconda/python2
    ```
5. Create a local tmp directory and install QIIME using conda
    ```bash
    mkdir ~/qiime_tmp
    conda create -n qiime1 python=2.7 qiime matplotlib=1.4.3 mock nose -c bioconda
    ```
    You will have to agree several times by typing `yes`. When the installation is finished run the following command to avoid a known bug.
    ```bash
    echo "backend: Agg" > ~/.config/matplotlib/matplotlibrc
    ```
6. Test the QIIME installation as follows
    ```bash
    source activate qiime1
    print_qiime_config.py -t
    ```
    Activating Qiime will take about 10 seconds. Running the test will take 30-60seconds. You should be greeted by (or similar) output:
    ```
    Ran 9 tests in 0.413s

    OK
    ```   
7. Copy the following folder, which contains the data and example scripts for you to modify.
    ```bash
    cp ~gavincbm/* ~/ssc
    ```
8. Change into the `ssc` directory and copy and rename the example scripts as follows
    ```bash
    cd ssc
    cp serial.pbs split.pbs
    cp paralel.pbs otu.pbs
    cp paralel.pbs div.pbs
    ```
9. Modify the scripts as follows using the command line editor `vim`. `vim` is not 100% user-friendly: when you first enter it, you need to hit `i` to change into insert mode to edit the file. When done editing, hit `ESC` and type `:wq` to write (save) and quit out of the file. If you made mistakes and don't want to save the file, you can use `:q!` to quit out of the file.
    ```
    vim split.pbs
    ```
    Edit the file to match the following below (you can choose a different job name for `#PBS -N`):
    ```bash
    #!/bin/bash --login
    #PBS -l walltime=01:00:00
    #PBS -l select=1:ncpus=2
    #PBS -N 2017_18_ssc_splitting_libraries
    #PBS -A d411-training
    # the following command changes the directory to the directory that the job was submitted from
    cd $PBS_O_WORKDIR
    # loading miniconda2
    module load miniconda/python2
    # activating virtualenv
    source activate qiime1
    # set temp_dir
    export TMPDIR=~/qiime_tmp
    # splitting libraries
    echo "splitting libraries"
    time split_libraries_fastq.py -m map.tsv -i read1_04012018.fastq.gz -b index_04012018.fastq.gz -o ./slout -q 19 --rev_comp_barcode --rev_comp_mapping_barcodes
    # deactivating virtualenv
    source deactivate
    # submit parallel second batch script
    qsub otu.pbs
    ```
    ```
    vim otu.pbs
    ```
    Edit the file to match the following below (you can choose a different job name for `#PBS -N`):  
    ```bash
    #!/bin/bash --login
    #PBS -l walltime=01:00:00
    #PBS -l select=1:ncpus=32
    #PBS -N 2017_18_ssc_picking_otus
    #PBS -A d411-training
    # the following command changes the directory to the directory that the job was submitted from
    cd $PBS_O_WORKDIR
    # loading miniconda2
    module load miniconda/python2
    # activating virtualenv
    source activate qiime1
    # set temp_dir
    export TMPDIR=~/qiime_tmp
    # picking OTUs
    echo "Picking OTUs with open reference"
    time pick_closed_reference_otus.py -i slout/seqs.fna -o otus -a -O 16
    # deactivating virtualenv
    source deactivate
    ```
10. Submit the first script via `qsub split.pbs`. Wait. Status of the job can be checked with `qstat`. `split.pbs` will submit the second job `otu.pbs` automatically.
11. Once the scripts have finished, check the the number of assigned OTUs and make note the samples with the least number of counts. You might have to log onto Cirrus again and load QIIME again.
    ```bash
    ssh username@login.cirrus.ac.uk
    module load miniconda/python2
    source activate qiime1
    ```
    ```bash
    biom summarize-table -i ./otus/otu_table.biom
    ```
    ```bash
    # output
    ssc.17.18.57: 23961.0
    ssc.17.18.62: 32224.0
    ssc.17.18.64: 35409.0
    ...
    ssc.17.18.52: 175321.0
    ssc.17.18.44: 222190.0
    ssc.17.18.69: 257628.0
    ```
12. Write another batch script to analyse sample diversity. The `-e` parameter determines the sampling depth. If `e` is set larger than a samples OTU count, it won't be included in the analysis. It is thus a tradeoff of analysis quality and amount of samples in the analysis. In the example output above the lowest count was `23961` and was used to set `e` in the script below.
    ```bash
    vim div.pbs
    ```
    Edit the file to match the following below. You can choose a different job name for `#PBS -N`. Adjust `e` to the lowest output obtained in your output from the OTU-table:
    ```bash
    #!/bin/bash --login
    #PBS -l walltime=01:00:00
    #PBS -l select=1:ncpus=32
    #PBS -N 2017_18_ssc_core_diversity
    #PBS -A d411-training
    # change to the directory that the job was submitted from
    cd $PBS_O_WORKDIR
    # loading miniconda2
    module load miniconda/python2
    # activating virtualenv
    source activate qiime1
    # setting temp_dir
    export TMPDIR=~/qiime_tmp
    # Analysing core diversity
    echo "Analysing core diversity"
    time core_diversity_analyses.py --recover_from_failure -o cdout -i otus/otu_table.biom -m map.tsv -t otus/97_otus.tree -e 23961
    # deactivating virtualenv
    source deactivate
    ```
    ```bash
    qsub div.pbs
    ```
    Should the script not run to completion (e.g. no `index.html` in the `cdout` folder), just re-submit the `div.pbs` script. The `--recover_from_failure` should allow it to take off from where it failed.
13. Once finished, open a local terminal and copy the output directory to your local machine to interrogate the data.
    ```bash
    scp -r cirrus:~/ssc/cdout/ ./cdout/
    ```
    The `index.html` file provides access to taxa-tables and bar-charts, alpha diversity rarefaction plots and 3D PCoA plots of sample distances.
