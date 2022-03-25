
# NIPM Singularity Containers



1. [Singularity](#Sec_intro)
    + [Singularity container use on val](#Sec_container_use)
    + [Singularity: container software](#Sec_container_software)
2. [Shell into an instance](#Sec_singularity_instance)
3. [Run with the exec command](#Sec_singularity_exec)

## 1. Singularity {#Sec_intro}

Singularity is a software application that allow users to have ‘full control’ over their operating system without the need for any ‘super-user’ privileges using the notion of containers. Singularity can give the user the freedom they need to install the applications, versions, and dependencies for their workflows without impacting the system in any way. For more detailed documentation on singularity see <https://singularity-docs.readthedocs.io/en/latest/>.

### 1.1 Singularity container use on val {#Sec_container_use}

Atty and Val are the compute clusters for NIPM.
Atty (atty.nipm.unlv.edu) is the storage server. It has 5 55TB disks and additional storage clusters.
Val (val.nipm.unlv.edu) is the compute server. It has 8 nodes with 80 cores on each node. 

![](NIPM_singularity/valsetup.png)

### 1.2 Singularity: container software {#Sec_container_software}

When running jobs on val, we can use singularity containers. I have shared two container images under `/data2/han_lab/singularity`.
`py37.ml.mkl.sif`: includes python3.7, sklearn, and other machine learning related software.
`rnaseq.sif`: includes fastqc, samtools, STAR, stringtie, hisat, and many more RNA-seq related software. 
The complete list of software can be found in the files `py37.ml.list` and `rnaseq.list`.

**It is important that you copy the image over to the node you plan to run jobs on and invoke the local version of the image that you copied over.** Having the software on atty and running them on val nodes leads to unnecessary network IO that significantly slows the jobs. Network IO between val nodes and atty is one of the most serious bottlenecks of performance. Do not invoke the container that is residing on atty directly. 

Another important thing to note is that the directories starting with `/data2` or `/data4` are atty attached disks that are mounted on val. Your home directory which is on `/data1/home/XXX` are also actually residing on atty. So **copying them over to your home directory or any directory under `/data2` or `/data4` would be defeating the purpose of copying since you would be copying it to another place within atty.** You have to copy it over to a local disk attached to the val nodes. One place you can use is `/scratch/*`. Another place is your home directory which has a 20G quota.   

For example, to use the py37.ml.mkl container on node02
```bash
ssh node02
cp /data2/han_lab/singularity/py37.ml.mkl.sif /scratch/han_lab/
```

Once you have the image on a local space on one of the nodes, you can run the image.
There are two ways to run the image. 

## 2. 'Shell' into an 'instance' {#Sec_singularity_instance}

You can make an 'instance' that you can shell into and out of like any other machine. This instance will stay running persistently as you log out and log into the nodes, and you can shell into the machine multiple times and check on the jobs progress. 

Make sure to bind all the atty disks that you will need read/write access to with the `--bind` option. 
```bash
singularity instance start --bind /data1,/data2,/data4,/scratch /scratch/han_lab/py37.ml.mkl.sif node02
singularity instance list
singularity shell --writable instance://node02
```

```bash
$ singularity instance start --bind /data1,/data2,/data4,/scratch /scratch/han_lab/py37.ml.mkl.sif node02
INFO:    instance started successfully
$ singularity instance list
INSTANCE NAME    PID      IP              IMAGE
node02           9938                     /scratch/han_lab//py37.ml.mkl.sif
$ singularity shell --writable instance://node02
(py37.ml.mkl) Singularity> 
```

Once you are inside the container after the shell command, you will see a different prompt.
`(py37.ml.mkl) Singularity>`

After that it is the same as running any other linux environment.
Once you have finished running your jobs, **remember to stop the instance and remove your copy of the container.** List the name of your instances and stop them with the commands below. 

```bash
singularity instance list
singularity instance stop node02
```

## 3. Run with the exec command {#Sec_singularity_exec}

You can run any kind of software that is part of the container using the exec command. Here are some examples. 

Running hisat2:
```bash
$ singularity exec rnaseq.sif hisat2
No index, query, or output file specified!
HISAT2 version 2.2.0 by Daehwan Kim (infphilo@gmail.com, www.ccb.jhu.edu/people/infphilo)
Usage: 
  hisat2 [options]* -x <ht2-idx> {-1 <m1> -2 <m2> | -U <r>} [-S <sam>]
```

Running samtools:
```bash
$ singularity exec rnaseq.sif samtools
Program: samtools (Tools for alignments in the SAM format)
Version: 1.9 (using htslib 1.9)

Usage:   samtools <command> [options]
```

Running bedtools:
```bash
$ singularity exec rnaseq.sif bedtools
bedtools is a powerful toolset for genome arithmetic.

Version:   v2.29.2
About:     developed in the quinlanlab.org and by many contributors worldwide.
Docs:      http://bedtools.readthedocs.io/
Code:      https://github.com/arq5x/bedtools2
Mail:      https://groups.google.com/forum/#!forum/bedtools-discuss

Usage:     bedtools <subcommand> [options]
```

Running bedtools intersect:
```bash
$ singularity exec rnaseq.sif bedtools intersect
Tool:    bedtools intersect (aka intersectBed)
Version: v2.29.2
Summary: Report overlaps between two feature files.

Usage:   bedtools intersect [OPTIONS] -a <bed/gff/vcf/bam> -b <bed/gff/vcf/bam>
```

You can also give it all the arguments you would give a regular program. It is important that you bind the disks that contain your data, so the container can see your data. But, of course, it is better if you copy your data over to `/scratch` instead. 

```bash
$ singularity exec --bind /scratch,/data4 rnaseq.sif bedtools sort -i /data4/han_lab/isoforms/BRCA/SRR4306964/SRR4306964.gtf
```

The beauty of this approach, in addition to all the gain in speed you get from having your software on the local /scratch space, is that you can use almost any bioinformatics software of any version you want, without maintaining a working environment. 

At  <https://quay.io/biocontainers/>, they provide more than 6000 bioinformatics software in a dockerized form that you can just pull to your node /scratch space with a singularity command. [If the software is on bioconda, it is on quay.io](https://biocontainers-edu.readthedocs.io/en/latest/conda_integration.html). 

For example, to pull the salmon container from `quay.io`:
```bash
$ singularity pull docker://quay.io/biocontainers/salmon:1.2.1--hf69c8f4_0
INFO:    Converting OCI blobs to SIF format
WARNING: 'nodev' mount option set on /tmp, it could be a source of failure during build process
INFO:    Starting build...
Getting image source signatures
Copying blob a3ed95caeb02 done  
Copying blob 77c6c00e8b61 done  
Copying blob 3aaade50789a done  
Copying blob 00cf8b9f3d2a done  
Copying blob 7ff999a2256f done  
Copying blob d2ba336f2e44 done  
Copying blob dfda3e01f2b6 done  
Copying blob a3ed95caeb02 skipped: already exists  
Copying blob 10c3bb32200b done  
Copying blob 4970ff4998a5 done  
Copying config a9e7773159 done  
Writing manifest to image destination
Storing signatures
2021/10/01 15:47:30  info unpack layer: sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
2021/10/01 15:47:30  info unpack layer: sha256:77c6c00e8b61bb628567c060b85690b0b0561bb37d8ad3f3792877bddcfe2500
2021/10/01 15:47:30  warn rootless{dev/console} creating empty file in place of device 5:1
2021/10/01 15:47:31  info unpack layer: sha256:3aaade50789a6510c60e536f5e75fe8b8fc84801620e575cb0435e2654ffd7f6
2021/10/01 15:47:31  info unpack layer: sha256:00cf8b9f3d2a08745635830064530c931d16f549d031013a9b7c6535e7107b88
2021/10/01 15:47:31  info unpack layer: sha256:7ff999a2256f84141f17d07d26539acea8a4d9c149fefbbcc9a8b4d15ea32de7
2021/10/01 15:47:31  info unpack layer: sha256:d2ba336f2e4458a9223203bf17cc88d77e3006d9cbf4f0b24a1618d0a5b82053
2021/10/01 15:47:31  info unpack layer: sha256:dfda3e01f2b637b7b89adb401f2f763d592fcedd2937240e2eb3286fabce55f0
2021/10/01 15:47:31  info unpack layer: sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
2021/10/01 15:47:31  info unpack layer: sha256:10c3bb32200bdb5006b484c59b5f0c71b4dbab611d33fca816cd44f9f5ce9e3c
2021/10/01 15:47:31  info unpack layer: sha256:4970ff4998a5ab9fad4baf0b2e71ce6349518e48ef330d683dc77fc80f176061
INFO:    Creating SIF file...
$ 
$ ls
mhan  R.3.6.3.phylo.sif  R.4.0.2.Bioc.sif  rnaseq.sif  salmon_1.2.1--hf69c8f4_0.sif  
```

Now, you can run salmon immediately on your data, without worrying about installing dependencies, etc. 

```bash
$ singularity exec --bind /scratch salmon_1.2.1--hf69c8f4_0.sif salmon quant
WARNING: Skipping mount /var/singularity/mnt/session/etc/resolv.conf [files]: /etc/resolv.conf doesn't exist in container
Version Info: Could not resolve upgrade information in the alotted time.
Check for upgrades manually at https://combine-lab.github.io/salmon
    salmon v1.2.1
    ===============

    salmon quant has two modes --- one quantifies expression using raw reads
    and the other makes use of already-aligned reads (in BAM/SAM format).
    Which algorithm is used depends on the arguments passed to salmon quant.
    If you provide salmon with alignments '-a [ --alignments ]' then the
    alignment-based algorithm will be used, otherwise the algorithm for
    quantifying from raw reads will be used.

    to view the help for salmon's quasi-mapping-based mode, use the command

    salmon quant --help-reads

    To view the help for salmon's alignment-based mode, use the command

    salmon quant --help-alignment
$ cp /data2/han_lab/SRR1553607.fastq .
$
$ singularity exec --bind /scratch,/data2 salmon_1.2.1--hf69c8f4_0.sif salmon quant -i /data2/han_lab/stevepark/sad-0.0.1/salmon_index/ -l A  -r SRR1553607.fastq -o out
WARNING: Skipping mount /var/singularity/mnt/session/etc/resolv.conf [files]: /etc/resolv.conf doesn't exist in container
Version Info: Could not resolve upgrade information in the alotted time.
Check for upgrades manually at https://combine-lab.github.io/salmon
### salmon (mapping-based) v1.2.1
### [ program ] => salmon 
### [ command ] => quant 
### [ index ] => { /data2/han_lab/stevepark/sad-0.0.1/salmon_index/ }
### [ libType ] => { A }
### [ unmatedReads ] => { SRR1553607.fastq }
### [ output ] => { out }
Logs will be written to out/logs
[2021-10-01 15:57:29.433] [jointLog] [info] setting maxHashResizeThreads to 88
[2021-10-01 15:57:29.433] [jointLog] [info] Fragment incompatibility prior below threshold.  Incompatible fragments will be ignored.
[2021-10-01 15:57:29.433] [jointLog] [info] Usage of --validateMappings implies use of minScoreFraction. Since not explicitly specified, it is being set to 0.65
[2021-10-01 15:57:29.433] [jointLog] [info] Usage of --validateMappings implies a default consensus slack of 0.2. Setting consensusSlack to 0.35.
[2021-10-01 15:57:29.433] [jointLog] [info] parsing read library format
[2021-10-01 15:57:29.433] [jointLog] [info] There is 1 library.
```

Sometimes, you may have issues with the containers on `quay.io`, because these are built for docker and we are using them for singularity. But, most of the times, they work as is without issue.
This will cut down a lot of time you spend on installing and maintaining software environments.  
If you want to combine multiple software and make it into one container that contains all your pipeline, you can refer to tutorials such as here:
<https://pawseysc.github.io/containers-bioinformatics-workshop/3.pipeline/index.html>
But, it is generally recommended to keep the containers simple and limit it to the one software - one container mapping. 

Hanlab has a few containers with several commonly used software that can be found here that you are welcome to copy to `/scratch` and use immediately: `/data2/han_lab/singularity/`.

If you need to build your own container, you can write your script to install all the software you need in a definition `.def` file and then use the following command in a computer that you have root privileges (i.e. sudo).

Build an .sif image (rnaseq.sif) from the definition file:
```bash
sudo singularity build rnaseq.sif rnaseq.def
```
Once you have the singularity container, you can securely copy it over to val. 
