# Step-by-step instructions on Qiime data processing
1. Enter your meta data into the [mapping file on Google docs ](https://docs.google.com/spreadsheets/d/1AIN9XGXPXAuC5phEK36AEsDgUIEfAg_RY-ZbD36IssY/edit?usp=sharing) (the link can be found on Moodle).
2. Once the class has finished updating the spreadsheet, download the mapping file as tab-separated value file (File > Download as > Tab-separated values). Take note of where the file is in your file-system (e.g. Downloads folder) and rename the file to `map.tsv`.
3. Open two separate terminal windows. On Windows you may want to use GitBash or Putty. On a Mac, you can use the terminal application. You will use one terminal to work remotely on Cirrus, the other to work on your local machine.
4. Next we will log onto Cirrus. Your Cirrus username and password can be found on the [Cirrus SAFE (https://www.archer.ac.uk/tier2/)](https://www.archer.ac.uk/tier2/). Your Cirrus password is different from the SAFE password. To find the Cirrus password mouse-over `Login-accounts` in the navigation menu and then click on `username@Cirrus`. On the bottom of the next page, click on `View Login Account Password`. After entering your SAFE password it will reveal your Cirrus password. Copy the password into your clipboard (`Cmd/Ctrl+C`).

    On the terminal `secure shell` connect to Cirrus. The password prompt cursor won't move as you type your password. On the GitBash you can paste your password from the clipboard with a right-click. On the Mac termianl you can use `Cmd+C`.

    **Important:** do not enter your password repeatedly incorrectly, or you and your colleagues might be locked out.

    ```bash
    ssh username@login.cirrus.ac.uk
    ```
    If you are logging in for the first time, you will be immediately asked to change your password. First re-type or re-paste the password you got of the SAFE. Then enter a password of your choosing. You will be prompted to do so twice. Your password cannot be a word found in the dictionary.
5. On Cirrus, create a working directory in your home directory, e.g `ssc`.
    ```bash
    mkdir ~/ssc
    ```
    The tilde (`~`) refers to your home directory. A quick reminder from your command line workshop:
    * `ls` lists the contents of a directory
    * `cd foldername` changes the directory to the folder or path entered (just like double-clicking on a folder icon). `cd` just by itself will always move you back to your home directory (`~`), `cd ..` will move you up one folder to the parental directory.
    * Once you type the first couple of letters of a file or path you can use `Tab` to autocomplete, or hit double `Tab` to get suggestions.
    * `pwd` will print the path of the current directory
    * `mv source destination` will move a file from one location to another (and rename it if you change the destination name).
    * `cp source destination` will create a copy of a file.
6. On your other terminal window, navigate to the location of your `map.tsv` file; e.g. `cd Downloads` if the file is located there.

    Using `secure copy (scp)` upload the file to Cirrus. The format of any copy operation is always origin to destination.
    ```bash
    scp map.tsv username@login.cirrus.ac.uk:~/ssc/

    ```
7. On the Cirrus terminal load the `miniconda/python2` module. This will load a version of Python, which has been installed centrally on Cirrus.
    ```bash
    module load miniconda/python2
    ```
8. Create a local temporary directory. Later, in the job scripts we will add a command to set QIIME to use this temporary directory.
    ```bash
    mkdir ~/qiime_tmp
    ```
9. Install QIIME using the miniconda package manager `conda`. You will have to agree to the installation by typing `yes`. The package manager will potter about downloading the many programmes and dependencies that make up QIIME. Be patient, your command prompt won't react to any commands.
    ```bash
    conda create -n qiime1 python=2.7 qiime matplotlib=1.4.3 mock nose -c bioconda
    ```
10. Activate QIIME to test the installation. QIIME has been installed in a virtual environment, which allows developers to precisely control the programme versions used for their software. This avoid conflicts with any central and potentially incompatible software.
    ```bash
    source activate qiime1
    print_qiime_config.py -t
    ```
    Activating Qiime will take a couple of sections. Running the test might also take a while. You should be greeted by a whole wall of text, the last couple of lines read similar to the following.
    ```
    Ran 9 tests in 0.413s

    OK
    ```
11. After the test, execute the following command to avoid a known bug when executing QIIME on a server without a display (also known as a headless system).
    ```bash
    echo "backend: Agg" > ~/.config/matplotlib/matplotlibrc
    ```
12. Copy the contents of the following folder, which contains the data and example scripts for you to modify. The asteriks `*` wildcard matches any file name in the `~gavincbm` folder.
    ```bash
    cp ~gavincbm/* ~/ssc
    ```
13. Change into the `ssc` directory and copy and rename the example scripts as below. `paralel.pbs` was misspelled on the server.
    ```bash
    cd ssc
    cp serial.pbs split.pbs
    cp paralel.pbs otu.pbs
    cp paralel.pbs div.pbs
    ```
14. Next we will work on job-submission scripts. Remember from your presentation by Dr. Gavin Pringle that job-queues on high-performance compute clusters are necessary to distribute load and harness the compute power optimally.

    Once a job has been submitted to the job-queue on Cirrus, you could log off and even shut-down your PC (don't do this now!).

    Modify the scripts as follows using the command line editor `vim`. `vim` is not 100% user-friendly:
    * When you first enter it, you need to hit `i` to change into insert mode to edit the file.
    * When done editing, hit `ESC` and type `:wq` to write (save) and quit out of the file (you can see what you are typing in the lower left corner of the terminal).
    * If you made mistakes and don't want to save the file, you can use `:q!` to quit out of the file.

    Start by editing the first job script.
    ```
    vim split.pbs
    ```
    Edit the file to match the content below. You can choose a different job name for `#PBS -N`. Comments (starting with `#`, save for the first 5 lines) aid in the legibility and serviceability of code, but aren't executed.
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
15. The first job-script `split.pbs`, which uses only a single CPU core, will submit the second script `otu.pbs`, which uses 16 CPU cores. As single-core jobs are run for free on Cirrus, this makes most optimal use of our core-hour budget.

    Hence, before submitting `split.pbs` to the job-queue, you will have to write the second script as well.
    ```
    vim otu.pbs
    ```
    Edit the file to match the content below. You can choose a different job name for `#PBS -N`.
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
16. Submit the first script to the job-queue via `qsub split.pbs`. Your entry will be confirmed by a job-number; e.g. `201460.indy0-login`.
17. You can check your job's status with the `qstat` command. `qstat` by itself will list all jobs currently in the queue or running. Job's in the queue are labelled with `Q` and `R` if they are being executed and `E` or `F` if there is an error.

    To only see your jobs, you can use `qstat -u username` or add your job number to the command, e.g. `qstat 201460`.
18. Once a job has finished it will disappear from the `qstat` list, however you can check the job history by using `qstat 201460 -H`.

    Every job, will create two log-files: one for the standard output (this would include the echo statements and timing information) and one for the error output.

    These log-files will appear in the folder you submitted your job from (`~/ssc` in this case) and contain the job-name (`#PBS -N ...` in your batch script) followed by either `.o` or `.e` and the job-number; e.g. `2017_18_ssc34_split_libraries.o200541` and `2017_18_ssc34_split_libraries.e200541`.

    Should your job have failed, the error-log will provide information about what might have gone wrong. You can read the files using either `cat` to output the content to the standard out (e.g. `cat 2017_18_ssc34_split_libraries.e200541`) or using the programme `less` (e.g. `less 2017_18_ssc34_split_libraries.e200541`). Hit `q` to quit out of `less`.
19. The `split_libraries_fastq.py` programme in `split.pbs` will generate an output folder `slout`, which contains the demultiplexed and quality-filtered sequencing data.

    Similarly, `pick_closed_reference_otus.py` in `otu.pbs` will generate an output folder `otus`, which contains the OTU-table.
20. Whilst the first two jobs are running, you can start on the third job-script. You can only complete this script, once the second job-script has finished. You will have to add a number after `-e`. More on this later.
    ```bash
    vim div.pbs
    ```
    Edit the file to match the following below. You can choose a different job name for `#PBS -N`.
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
    time core_diversity_analyses.py --recover_from_failure -o cdout -i otus/otu_table.biom -m map.tsv -t otus/97_otus.tree -e
    # deactivating virtualenv
    source deactivate
    ```
21. Once the first two jobs have finished, you will find both an `slout` and `otus` directory in your working directory `ssc`. If you were logged out of Cirrus, log in again and re-activate the QIIME environment.
    ```bash
    ssh username@login.cirrus.ac.uk
    module load miniconda/python2
    source activate qiime1
    ```
    In order to complete the `div.pbs` job-script, check the the number of assigned OTUs.
    ```bash
    biom summarize-table -i ./otus/otu_table.biom
    ```
    You will receive a list of all 42 samples in the analysis and how many OTUs were assigned to each of them.
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
    Make note of the sample with the least number of counts.
22. Open `div.pbs` again and set the `-e` parameter on the line of the `core_diversity_analyses.py` script to the smallest number of OTU counts. If `e` is set larger than a sample's OTU count, it won't be included in the analysis. It is thus a tradeoff of analysis quality and amount of samples in the analysis. In the example output above the lowest count was `23961` and was used to set `e`.
23. You can now submit `div.pbs` to the job-queue.

    If you didn't carry out step 11 at the correct point, you may also receive a DISPLAY error. Re-execute the code in step 11 and re-submit `div.pbs`.

    Should the script not run to completion for other reasons (no `index.html` in the `cdout` folder), just re-submit the `div.pbs` script. The `--recover_from_failure` should allow it to take off from where it failed.
24. Once `div.pbs` has finished, you can copy its output to your local machine to interrogate. On your other (local) terminal use secure-copy to download the data.
    ```bash
    scp -r username@login.cirrus.ac.uk:~/ssc/cdout ./cdout/
    ```
    The `index.html` file provides access to taxa-tables and bar-charts, alpha diversity rarefaction plots and 3D PCoA plots of sample distances.
