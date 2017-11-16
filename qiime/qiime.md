# A Quantitative Insights Into Microbial Ecology (QIIME) pipeline: From FASTQ to diversity analysis

This self-paced assisted learning tutorial covers a basic [QIIME](http://qiime.org/) workflow using data from an [Illumina MiSeq](https://www.illumina.com/systems/sequencing-platforms/miseq.html) sequencing run carried out in the academic year 2015-2016. In order to keep computing times reasonable, a subset of 50,000 sequences (by contrast to 15-25 million sequences of the full dataset) will be used.

Large parts of this SPLAT will be carried out remotely on a UCL server. As such, you should be familiar in using the `command line` to navigate the file structure and use `git` to download data. If you feel you need a refresher, please refer to the respective SPLATs.

## Getting access to UCL Aristotle

[_Aristotle_](https://wiki.rc.ucl.ac.uk/wiki/RC_Systems#Aristotle) is part of UCL's Research Compute Platform, or UCL's system of high performance compute clusters. Unlike 'traditional' HPC, _Aristotle_ is not fashioned with a [job queue](https://en.wikipedia.org/wiki/Job_queue). So, instead of having to wait for programmes to be submitted to a compute node, we can run our programmes interactively, just as you would do on your local terminal.

Once you access Aristotle, you will also notice that your home-directory (`~/`) is mapped to your central ISD filestore (`N:` drive), which has a quota of only 1GB.

As we are using a **shared and limited** computing resource, which is open to all UCL users, _Aristotle_ is to be used with consideration; e.g. don't run your full scale data analysis on _Aristotle_!

### Using secure shell (ssh)

On your command line (GitBash or Mac terminal), use the secure shell (`ssh`) to log onto _Aristotle_, as below, whereby `<username>` needs to be replaced with your UCL userid, such as `zcbcxxx`. Following the command you will be greeted with a prompt to enter your UCL password. Please note that the password cursor won't move when you type your password.

```bash
ssh <username>@aristotle.rc.ucl.ac.uk
# prompt will ask for your UCL user password, the cursor won't move as you type
```

Should you receive the following or a similar message, confirm it by typing `yes`. _Aristotle_ (in this case) will be added to the list of known hosts.

```
The authenticity of host 'aristotle.rc.ucl.ac.uk (<no hostip for proxy command>)' can't be established.
ECDSA key fingerprint is SHA256:DWhqTad2on4D7/u6STVPXxLtC4/ZakhbZZhYmlWMJvc.
Are you sure you want to continue connecting (yes/no)?
```

Whilst you are on Aristotle, your command prompt, the first couple of letters preceeding every line, will change to `-bash-4.2$`. You can exit any remote server any time through the `exit` command.

For security reasons, _Aristotle_ cannot be access from outside the UCL network (you will received a timeout error). In this case, you can either install software to access the [UCL virtual private network](http://www.ucl.ac.uk/isd/services/get-connected/remote-working/vpn), or `ssh` connect to UCL _Socrates_ first and then connect from the _Socrates_ shell to _Aristotle_.

```bash
ssh <username>@socrates.ucl.ac.uk
# prompt will ask for password
ssh <username>@aristotle.rc.ucl.ac.uk
# prompt will ask for password again
```

### Adding ssh keys to the server

In order to avoid continuously having to type your password whenever you log onto remote servers, **secure shell keys** can be added to the server for password-less authentication. You have previously generated keys using the `ssh-keygen` command (p4, Git SPLAT) to allow `git` to communicate with a GitHub repository. The keys are stored in the folder `~/.ssh`.

The keys can be copied to the server the 'old-fashioned' way ...

```bash
cat ~/.ssh/id_rsa.pub | ssh <username>@aristotle.rc.ucl.ac.uk "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
# the -p argument for mkdir makes sure this command won't fail in case the directory already exists
# && allows us to chain separte commands together, whilst >> appends the redirected output to a file
```
... or using `ssh-copy-id`.

```bash
ssh-copy-id <username>@aristotle.rc.ucl.ac.uk
```

In both cases you will have to type in your UCL account password one more time.

If you are outside the UCL network, you can do the same thing for _Socrates_, then generate ssh keys there and copy these to _Aristotle_.

```bash
# on local terminal
ssh-copy-id <username>@socrates.ucl.ac.uk
# on socrates terminal
ssh-keygen -t rsa
# as socrates does not support ssh-copy-id
cat ~/.ssh/id_rsa.pub | ssh <username>@aristotle.rc.ucl.ac.uk "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```

### Creating an ssh config

Typing all these ssh-commands can become arduous quickly. Let's make it easier, by creating an **ssh config**. In `~/.ssh/`, create a text-file called `config`. There are several ways to do this, but one of the fastest is to use the editor `vim`.

```bash
cd ~/.ssh/
vim config
```

Using vim, in the editor hit `i` to change to `insert` mode, then type the text below.

```vim
Host aristotle
   User <username>
   HostName aristotle.rc.ucl.ac.uk
```

Don't forget to indent the lower two lines. When you finished typing, hit `Esc` followed by `:wq` to write and quit (you can see what you are typing in the lower left corner of the terminal window).

You should now be able to `ssh aristotle` without having to add username and the full server address. The `Host` name can be freely picked.

If you are off campus, the ssh routing via _Socrates_ can be directly incorporated in a `proxyCommand`.

```vim
Host aristotle_remote
   User <username>
   HostName aristotle.rc.ucl.ac.uk
   proxyCommand ssh -W aristotle.rc.ucl.ac.uk:22 <username>@socrates.ucl.ac.uk
```

Now there is no more need separately to connect to _Socrates_ first; `ssh aristotle_remote` should do the trick.

### Secure copy (`scp`)

With `scp` you can copy files securely across an `ssh` connection. Local file(s) to a remote location, in this case _Aristotle_.

```bash
# a single file
scp /path/to/source-file <username>@aristotle.rc.ucl.ac.uk:/path/to/destination-folder/
# multiple files
scp file1.txt file2.txt file3.txt <username>@aristotle.rc.ucl.ac.uk:/path/to/destination-folder/
# or if the files are in different locations, just include the path
scp /path/to/file1.txt /path/to/file2.txt /path/to/file3.txt <username>@aristotle.rc.ucl.ac.uk:/path/to/destination-folder/
# Copy a folder and all its contents (-r for recursive)
scp -r /path/to/source-folder <username>@aristotle.rc.ucl.ac.uk:/path/to/destination-folder/
```
To copy file(s) from a remote server to your current local server, use the following syntax.

```bash
scp <username>@aristotle.rc.ucl.ac.uk:/path/to/source-file /path/to/destination-folder
```

Because you generated an `ssh config` and added your `ssh-key` to the server, the syntax can be significantly shortened and you won't have to type passwords.

```bash
scp /path/to/source-file aristotle_remote:/path/to/destination
```

Please note: unless you local server (i.e. your computer) can be reached via an IP-address, you cannot `scp` off the Aristotle shell; you always have to initiate the copying process off your local terminal.

## Processing and analysing soil sample data using QIIME

[QIIME](http://qiime.org/) (pronounced _chiime_, Quantitative Insights into Microbial Ecology) is a bioinformatics pipeline tool to perform microbiome analysis, starting with sequencing raw data.

The data was obtained in the academic year 2015-16. Four soil samples were collected from Gordon Square Gardens in London and genomic DNA was isolated. Using universal primers, the variable region 4 of 16S ribosomal RNA genes was amplified and sequenced on an Illumina MiSeq platform. A virtually identical workflow can be applied to analysing the microbiome of other niches, such as human skin.

Of the 15 million sequences obtained, 50,000 were randomly sampled to keep calculation times within reasonable time frames for this exercise setting.

### Downloading the exercise dataset

If you are not already, secure shell connect to _Aristotle_. Using `git`, download a copy of the exercise data.

```bash
git clone https://github.com/Cistron/bioc3301_50k ~/soil_50k --depth 1
## --depth 1 will only download the last commit of the respository
```

This will start to download files into the folder `soil_50k`. Depending on the internet connection the unpacking might take up to a few minutes; so do not fret if `git` is seemingly stuck.

Inside the the folder (`cd soil_50k`, `ls`) you will find several files, including `map.tsv`, a tab-separated value table containing information about the samples, including the barcodes, coordinates, etc.

`read1.fastq.gz` and `barcode.fastq.gz` contain the actual sequences in [FASTQ format](https://en.wikipedia.org/wiki/FASTQ_format) and are both compressed with gzip. In order to interrogate the files on the command, you can temporarily unzip them and pipe the output, e.g. `gunzip -c <filename> | less`.

The sequences in `read1.fastq.gz` are the first read from the [Illumina MiSeq sequencing](https://www.youtube.com/watch?v=womKfikWlxM) run and are 250bp long. `barcode.fastq.gz` contains the index/barcode sequences, which are later used to assign sequences to specific samples.

Note that both these files contain 50,000 sequences and are 200,000 lines long (you can test this wih `gunzip -c <filename> | wc -l`), though this is still less than 0.3% of the full dataset with more than 15 million sequences.

### "Installing" QIIME

As _Aristotle_ is part of UCL Research Compute Platform it has access to a whole range of centrally installed software. The list of available modules can be queries using `module avail <query-term>`. Querying for Python will reveal a whole range of different Python versions.

`module avail python`

```bash
------------------------------------ /shared/ucl/apps/modulefiles/development -------------------------------------
python/2.7.12          python/3.4.3           python/3.6.1/gnu-4.9.2
python/2.7.9           python/3.5.2           python/3.6.3

-------------------------------------- /shared/ucl/apps/modulefiles/bundles ---------------------------------------
python2/recommended          python3/3.5                  python3/recommended(default)
python3/3.4                  python3/3.6
```

QIIME has previously been installed and added to the `python2/recommended` stack. `module show python2` will reveal two URLs on GitHub, which list all libraries and packages that are part of the `python2` module. `module load python2` will load the module, but won't prompt any feedback.

If you want to install QIIME locally on your computer, [the developers provide detailed instructions](http://qiime.org/install/install.html) on how to do this with the Python distribution `Miniconda`. Unfortunately, on Windows PCs you still need to use a somewhat cumbersome virtual machine.

Once QIIME is loaded/installed, you need to activate the virtual environment. A virtual environment is a quasi-copy of Python, which allows developers to curate very specific version combinations of Pyton libraries.

```bash
source activate qiime1
```

You should notice your command prompt change again. Depending on the setup the prompt will be preceeded by `(venv)` or the name of the virtual environment `(qiime1)`. You can close the virtual environment (on _Aristotle_) with `deactivate`.

In order to validate that all QIIME programmes are working appropriately, we can test the installation by running `print_qiime_config.py -t`. The script will output a list of dependency versions and finish with test results.

```
QIIME base install test results
===============================
.........
----------------------------------------------------------------------
Ran 9 tests in 0.069s

OK
```

### Microbiome analysis

#### Validate mapping file

The **mapping file** which contains all the sample metadata, including the sample barcodes, which are quasi unique sample identifiers. As Excel in particular produces broken tab-seperated-value (`.tsv`) files and errors in barcodes (such as duplications) sometimes happen, it is a good idea to check the mapping files for mistakes.

Don't forget to change directory into the data directory (`cd ~/soil_50k`), or your file paths will be off.

```bash
# making sure map file is correct, faults will be highlighted in maps.tsv.html
# excel doesn't produce proper .tsv files, so use e.g. Google documents to generate a map-file
validate_mapping_file.py -m map.tsv -o ./vmf
```

You should receive immediate feedback on the command line standard out (i.e. what you read on the screen); this message will also be logged in the text-file `./vmf/map.tsv.log`. Any mistakes will be highlighted in `./vmf/maps.tsv.html`. As _Aristole_ doesn't provide a graphical user interface, you will have to copy these files to your local machine using `scp`.

#### Demultiplexing and quality filtering

With growing capcities, modern high-throughput sequencing has departed from splitting samples onto separates physical lanes and the use of barcodes allows the muliplexing (i.e. mixing) of thousands of samples on a single sequencing flow cell. This works, as both sequence and barcode are on the same piece of DNA that is anchored to the flow-cell surface. For a refresher watch the [Illumina animation on sequencing by synthesis](https://www.youtube.com/watch?v=womKfikWlxM). During demulitplexing, barcodes, and thus sample identities, are re-assigned to specific sequences.

Based on sequence quality scores included in the FASTQ files, the sequences are also filtered or truncated (see [Bokulich et al, 2013](10.1038/nmeth.2276) for a deeper discussion of the topic).

Illumina's exact method for the calculation of quality scores is a closely guarded secret, but largely relies on adding the known bacteriophage **PhiX** genome to the sequencing runs. Error rates and quality scores can then be extrapolated/estimated/calculated for the target sequences based on the errors observed in the PhiX sequence.

```bash
split_libraries_fastq.py --barcode_type 12 -i read1.fastq.gz -m map.tsv -o ./slout -b barcode.fastq.gz
# http://qiime.org/scripts/split_libraries_fastq.html
```

`./slout/split_library_log.txt` provides a breakdown of the demultiplexing and filtering process. The higher the quality of the sequencing data, or the less aggressive the quality filtering, the less sequences are lost. The retained sequences are now in [FASTA format](https://en.wikipedia.org/wiki/FASTA_format) in `./slout/seqs.fna`.

Each FASTA header now contains the barcode assigned to the sequence. As with all text files, you can interrogate the file contents with `less` (`q` to quite out of it again) or `head` to print the first couple of lines (I would recommend caution with `cat` with such large text-files).

`count_seqs.py -i ./slout/seqs.fna` provides a brief summary of the number of sequences in the `.fna` file. How does this compare to the line-count (`wc -l`)?

#### Picking operational taxonomical units (OTU)

OTUs are proxies for microbial "species", which are determined by sequence identitiy of a marker gene - in our case 16S rRNA. Here we pick OTUs based on the Greengenes reference database. Sequences with more than or equal to 97% sequence identity will be assigned to a reference OTU.

For larger datasets, picking OTUs is computationally intensive and can be parallelised (use several CPU cores) with the `-a -O number_of_cores` parameters. Keep this mind when you run your full data analysis on HPC.

```bash
pick_closed_reference_otus.py -i ./slout/seqs.fna -o ./otus
# http://qiime.org/scripts/pick_closed_reference_otus.html
```

The main output is an OTU table, or in other words a table which lists how many times each OTU is observed in each sample. As this is a potentially very sparse table (lots of zeros), it is encoded in [BIOM format](http://biom-format.org/) and not human-readable. However, `biom summarize-table -i ./otus/otu_table.biom` provides a summary, including OTU counts per sample. Make a note of the smallest counts, as these will determine the sampling depth for further analyses.

#### Core diversity analyses

The `core_diversity_analyses.py` script calculates several diversity measures, including alpha diversity (the diversity within a sample), beta diversity (the diversity between samples), as well as taxa tables.

Several of the analyses require an equal number of sequences for each sample, which is determined by the sampling depth parameter `-e`. Samples with less sequences than the sampling depth will simply be excluded from the analysis; hence there is a tradeoff between the sampling depth and the number of samples in the analysis. In order to include all samples in the analysis, you could pick the lowest sequence count you saw OTU-table summary output.

```bash
core_diversity_analyses.py --recover_from_failure -o cdout/ -i otus/otu_table.biom -m map.tsv -t otus/97_otus.tree -e 1898
# http://qiime.org/scripts/core_diversity_analyses.html
# -e is the sampling depth, samples with less sequences will be excluded
# --recover_from_failure allows resuming analysis, should it crash
```

An overview of the results are compiled in an `index.html` file. Again, you will have to `scp` the output folder from your local machine in order to view these files, as _Aristotle_ doesn't provide a graphical user interface.

Of particular interest for your ongoing project on the skin microbiome are likely the 3D PCoA (principal coordinate analysis) plots ([PCa and PCoA explained](http://occamstypewriter.org/boboh/2012/01/17/pca_and_pcoa_explained/)). The weighted plot takes OTU abundance into account, which means rare OTU factor less into the distances between samples. How could PCoA plots be useful?

The QIIME website has a register of [all scripts](http://qiime.org/scripts/index.html), as well as [further tutorials](http://qiime.org/tutorials/index.html) that can lead you deeper down the rabbit hole.

Don't forget to `deactivate` your virtual environment and `exit` when you are finished.



<!--
* give them ideas for further data analysis?
	* Pose some questions
	* Point of the documentation for all the QIIME programmes
* use Silva instead of Greengenes? (http://qiime.org/home_static/dataFiles.html) -->
