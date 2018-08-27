# TypeSeq HPV
***NCI CGR laboratory HPV typing analysis workflows and R package***

TypeSeq HPV is an R package that includes  

* several helper functions for working with TypeSeq data  
* contains a "make" based pipeline for processing Ion or Illumina runs  
* contains a docker build file that includes all the dependencies inside a single container  
  
We recommend running the pipeline inside the docker container ```cgrlab/typeseqhpv:final_2018080604``` as it contains all the required dependencies in the correct locations.

The workflow manager we use is **drake** https://github.com/ropensci/drake


There are currently two main workflows each supporting either the Ion Torrent or Illumina NGS platforms.  Since TypeSeqHPV can be used on either platform we therefore have analysis for either. 

The only requirement for either workflow is either ```docker``` or ```singularity```



Ion Torrent Plugin
================

We also include a wrapper for the Ion Torrent server that can be uploaded via the provided zip file.  The prerequisite for running the Ion Torrent Plugin successfully is to install docker on the server ahead of time.

## Install Docker Community Edition On Torrent Server

Following instructions posted here https://docs.docker.com/install/linux/docker-ce/ubuntu/

### Uninstall previous version

It may be required to uninstall previous version of docker.  Skip this step if no previous version of docker installed.

```
sudo apt-get remove docker docker-engine docker.io
````

### Update Package Index and Install

````
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update

sudo apt-get install -y docker-ce
````

Try a hello world install test.
````
sudo docker run hello-world
````

### Post Installation Setup
  
Give plugin access to docker.
````
sudo usermod -aG docker [user name] 
sudo docker login (with any docker user id)
sudo usermod -aG docker ionian
sudo su ionian
docker login (with any docker user id)
su ionadmin
````

### Make Docker storage more robust on torrent server

Docker doesn't always clean up after itself.  Changing were docker keeps it's image layers will prevent critical partitions from filling up.  If this step is skipped after several runs of a docker plugin the Torrent Service job scheduler will stop working.

````
sudo service docker stop
cd /var/lib
sudo rsync -a docker /results/plugins/scratch/
sudo rm -rf docker
sudo ln -s docker /results/plugins/scratch/docker
sudo vi /etc/default/docker
````


## Download and add hpv-typing plugin zip file via torrent server gui

https://github.com/cgrlab/TypeSeqHPV/releases/download/2.1808.2701/TypeSeqHPV_TSv1_Ion_Torrent_Plugin.zip

Illumina Workflow
================
### 1. Generate fastq files from raw sequencing data using modified parameters
  
The Illumina fastq files need to be regenerated using the following custom bcl2fastq parameters:  
```  
bcl2fastq --runfolder-dir [location of run directory (local or networked drive) ]  
--output-dir [ any directory]  
--with-failed-reads  
--minimum-trimmed-read-length 11  
--mask-short-adapter-reads 11  
```  

This should generate two new fastq files, one for R1 and one for R2. Any version of bcl2fastq higher than  
2.19 should work.
  
### 2. Create New Directory for The Project  

The pre-built image, cwl file, and helper bash script are hosted in an archive on the NIH HPC at `https://hpc.nih.gov/~robersondw/TypeSeqerHPV_0.18.314.tar.gz`

The workflow will need to utilize a fresh working directory. Make sure that the contents of
`typeseqerHPV.0.18.31403.simg.tar.gz` are in this working directory.  The complete set of files needed for the task are:
  
  
   1. Bash script with execution command (in archive)
   2. Common workflow language file (in archive)
   3. The Singularity image (in archive)
   4. Pair of fastq files (provided by user)
   5. TypeSeqHPV run manifest (provided by user)
   6. Control definitions file (provided by user)
   
As an alternative the user can generate the singularity image by building from the Dockerfile.  The user will still need to download the latest cwl and shell script from this repository.     
    
### 3. Adjust the Shell Variables  

In the shell script there are 5 variables that will need to be entered manually to help with the Singularity
exec command. These include fastq1, fastq2, control_defs, run_manifest and working_dir.  
  
Note that the file paths specified should be relative to the working_dir so that the relationship is
maintained inside the container. Any paths relative to your local environment will not work inside
Singularity.
  
### 4. Run the Workflow  
  
These instructions were tested on the NIH HPC which uses Slurm Workload Manager. We used an
interactive node but the shell script can easily be adjusted for a batch job. For a fast completion time try
to use at least 16 or 24 cores and at least 60 GB RAM. A typical MiSeq run should complete in 45
minutes with 24 cores.
  
The workflow will generate a sub directory with the name of the cwl file and a time stamp (e.g.
Illumina_TypeSeqHPV-2018-03-15-185300.647).  

Inside this folder will be sub folders for each of the tools in the workflow. The Illumina_TypeSeqHPV* folder
and its sub directories can be considered temporary files and may be deleted as soon the workflow
finishes. This folder will need about 50 GB of disk space but will be reduced to about 2GB after the run
completes (these are typical sizes for MiSeq runs; NextSeq runs will require more space).  
  
The folder the end user should keep is called Illumina_TypeSeq_output and will be about 1 MB. This
will contain a report PDF and CSV tables.  

The version of Singularity we used to run the container is 2.4.4

