# MCScan (Python-version) Workflow on SCELSE Hadley-HPCC

by Mark Chan and Wee Soon Keong

5 October 2020

Using genomic islands from *Burkholderia thailandensis* AW34-19p (pink), AW34-19pp (purple) and E264 (ATCC) chromosome 1 as examples.

------

[TOC]



### 1. Generating `.gff` and `.faa` input files

##### 1.1 Identifying genomic islands using cumulative GC skew plot in python

For each bacteria strain, I generated one `.fa` output containing concatenated sequences of all detected genomic islands.  This can be input into a prokaryotic gene annotation software such as Prokka to generate `.gff` and `.faa` files:

> <img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201005172352861.png" alt="image-20201005172352861"  />
>
> ***Important***:
>
> If you have multiple nucleotide sequences in separate `fa` files, make sure to concatenate all `fa` files before using Prokka annotation. Do not run Prokka on multiple `fa` files and concatenate the `.gff` output. This is because conversion of `.gff` files to `.bed` format later on would only retain the first table in the `.gff` file. (If you concatenated `fa` files at the start, all annotations of separate sequences will be in the first/only/same table)
>
> **tl;dr** if you have multiple sequences in separate `fasta` files, concatenate them first before running on Prokka

##### 1.2 Obtaining files

Possible to annotate sequences using Prokka from [Galaxy](https://usegalaxy.eu/) server. Easier to view output files and is relatively fast.

Download general feature format `.gff` and amino acid fasta `.faa` files. Only these two file formats would be needed for MCScan (Python). 

### 2. Importing data

##### 2.1 Connecting to SCELSE Hadley HPCC

Connect to hadley.scelse.ntu.edu.sg using SSH client (PuTTY):

> IP address: 172.21.32.46
>
> Port: 22

Enter login credentials when prompted.

##### 2.2 Create a working directory

```shell
#go to directory
cd /gpfs1/scratch/keong/mark

#create directory and enter it
mkdir -p gi/bt/pinkpurple
cd gi/bt/pinkpurple
##The -p flag tells the mkdir command to create the main directory first if it doesn’t already exist
```

##### 2.3 Upload files using FileZilla

Login to FileZilla and access our working directory using remote site

```
/gpfs1/scratch/keong/mark/gi/bt/pinkpurple
```

Drag and drop the following files into the `pinkpurple` directory:

> pinkchr1.gff
>
> pinkchr1.faa
>
> purplechr1.gff
>
> purplechr1.faa

Check for successful upload of files

```shell
ls
```

### 3. Converting Prokka output files to MCScan input format

You can do this on Galaxy too, they have these file conversion functions available. Also allows you to view both files nicely in a tabular format.

Alternatively, we can convert the files locally as well (only takes a few seconds)

##### 3.1 Converting `gff` files to `bed` format

```shell
#loading jcvi module
module load jcvi/1.2.3

#converting .gff to .bed
python -m jcvi.formats.gff bed --type=CDS --key=ID pinkchr1.gff -o pinkchr1.bed
python -m jcvi.formats.gff bed --type=CDS --key=ID purplechr1.gff -o purplechr1.bed

##Variables
##--type: Feature type (third column of gff file; in this case its CDS. Alternatively you can input mRNA for transcriptomics analysis)
##--key: Prefix eof column 9 'Group' (in this case its ID)
##pinkchr1.gff is my input file in .gff format
##-o is output
##pinkchr1.bed is my output file name in .bed format
```

> Example `gff` file in a tabular format. *Note --type=CDS and --key=ID*:

![image-20200918132343486](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200918132343486.png)

> During the format conversion you should see something like this:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200918152447776.png" alt="image-20200918152447776" style="zoom: 80%;" />

```shell
#check for successful .bed file format conversion
ls
```

##### 3.2 Reformatting amino acid `fasta` files

```shell
#converting .faa to .cds
python -m jcvi.formats.fasta format pinkchr1.faa pinkchr1.cds
python -m jcvi.formats.fasta format purplechr1.faa purplechr1.cds

#then, converting .cds to .pep
cp pinkchr1.cds pinkchr1.pep
cp purplechr1.cds purplechr1.pep

#check for successful .pep file format conversion
ls
```

At this point, we would only need `.pep` and `.bed` files to proceed. We can remove all the other files.

```shell
#removing .gff .faa and .cds files
rm *.gff *.faa *.cds
```

### 4. Running pairwise synteny search on MCScan (Python)

##### 4.1 Creating PBS script

```shell
#create job submission script using nano text editor
nano submit.sh
```

Edit the following script accordingly and copy it into the terminal. Save and exit:

```
#!/bin/bash

# Run this as "qsub submit.sh"

# Set the resource requirements; 1 CPU, 5 GB memory and 5 minutes wall time.
#PBS -l select=1:ncpus=1:mem=8GB
#PBS -q std
#PBS -l walltime=00:05:00

# Name your Job
#PBS -N jcvi-test

# Load the necessary application

export PATH=$PATH:/gpfs1/scratch/keong/softwares/last-1080/src

module load jcvi/1.2.3



# Run your program.
cd ${PBS_O_WORKDIR}

python -m jcvi.compara.catalog ortholog BD5_chr1 AW17-22_chr1 --no_strip_names --dbtype prot --cscore=0.99
```

##### 4.2 Submitting job to Hadley

```shell
#submit PBS script as a job
qsub submit.sh
```

##### 4.3 Checking MCScan output files

```shell
#check for files
ls

#check for files and file sizes
ll
```

You should see MCScan output files in the same folder. The `.pdf` file size is a good indicator of whether the analysis worked and LaTeX was able to successfully generate the plot. You can visualize the plot by downloading the `pdf` file onto your local computer.

The `.last` file is raw LAST output, `.last.filtered` is filtered LAST output, `.anchors` is the seed synteny blocks (high quality), `.lifted.anchors` recruits new additional anchors to form the final synteny blocks.

### 5. Macrosynteny visualization

First, we generate a `.simple` file, which is a more succinct form than the `.anchors` file:

```shell
#generating .simple file from .anchors file
python -m jcvi.compara.synteny screen --minspan=30 --simple pinkchr1.purplechr1.anchors pinkchr1.purplechr1.anchors.new

#if got value error, try to reduce minspan to 10
```

Aside from `.bed` and synteny files we already have, we need to prepare 2 more additional files:

1. `seqids` file, which tells the plotter which genomic islands (or chromosomes/genes) to include, and
2. `layout` file, which tells the plotter where to draw what.

##### 5.1 Generating `seqids` file

For this analysis, pinkchr1 contains 8 genomic islands (named GI1-GI8) whereas purplechr1 contains 9 (named GI1-GI9). I want to look at all the genomic islands, so I will be including all of them. Note the order of the names in the `pinkchr1.purplechr1.anchors` file.

```shell
#generating seqids file
nano seqids
```

According to the naming order of the `anchors` file, the first line should contain pinkchr1 genomic islands (8 of them) and second line purplechr1 genomic islands (9 of them). Copy the following into the terminal. Save and exit:

```tex
GI1,GI2,GI3,GI4,GI5,GI6,GI7,GI8
GI1,GI2,GI3,GI4,GI5,GI6,GI7,GI8,GI9
```

***E.g. To invert GI1, just add a hyphen i.e. -GI1, -GI2, -GI3***

##### 5.2 Generating `layout` files

```shell
#generating layout file
nano layout
```

It is useful to know that the whole canvas is 0-1 on both x and y-axes. The first three columns specify the position of the track. Then rotation (in degrees), colour, label, vertical alignment (`va`), and then the genome `.bed` file. Track 0 is pinkchr1 (python starts reading from 0 #justpythings), and track 1 is purplechr1. We can leave the colour blank for now (it will be default).

The next stanza specifies what edges to draw between the tracks. `e, 0, 1` asks to draw edges between track 0 and 1, using information from the `.simple` file.

Copy the following into the terminal. Save and exit:

```tex
# y, xstart, xend, rotation, color, label, va,  bed
 .7,     .1,    .8,       0,      , BtPINKchr1, top, pinkchr1.bed
 .5,     .1,    .8,       0,      , BtPURPLEchr1, top, purplechr1.bed
# edges
e, 0, 1, pinkchr1.purplechr1.anchors.simple
```

> ***Important***:
>
> Make sure there is no space (` `)  at the start of each line below `# edges`.

##### 5.3 Plotting figure

Now we have our input files ready, we can plot.

```shell
#plotting
python -m jcvi.graphics.karyotype seqids layout
```

This generates a `karyotype.pdf` that we can download and visualize:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201005192742961.png" alt="image-20201005192742961" style="zoom: 67%;" />

Further customization is possible. We can play around with the positions, colour, labels etc. in the `layout` file. Instead of a M&S candy design, we can invert the order of the chromosomes on purplechr1 by editing the `seqids` file, and it should generate a figure that is visually more appealing:

**Big brains?**

```tex
GI1,GI2,GI3,GI4,GI5,GI6,GI7,GI8
GI9,GI8,GI7,GI6,GI5,GI4,GI3,GI2,GI1
```

At least that was what i thought.

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201005193422507.png" alt="image-20201005193422507" style="zoom: 67%;" />

Seems like I still need to go back to my raw fasta files and invert the sequence. Sad.

### 6. Macrosynteny getting fancy

##### 6.1 Highlighting a specific block

We can highlight a specific gene or even a particular set of genes by editing the `.simple` file. Download the `.simple` file and edit it using a text editor (e.g. Sublime Text).

Note that each line in the `.simple` file is a synteny block, with start and stop of pinkchr1, then start and stop of purplechr1, final two columns are score and orientation. Mine looks something like this:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201006120317023.png" alt="image-20201006120317023" style="zoom:80%;" />

We can highlight each synteny block individually with any colour abbreviation supported by matplotlib:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201006141218526.png" alt="image-20201006141218526" style="zoom:80%;" />

| **Character** | Colour  |
| :------------ | ------- |
| b             | blue    |
| g             | green   |
| r             | red     |
| c             | cyan    |
| m             | magenta |
| y             | yellow  |
| k             | black   |
| w             | white   |

Remember to save the updated changes before uploading the `.simple` file into FileZilla.

Lets plot:

```shell
#plotting
python -m jcvi.graphics.karyotype seqids layout
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201006141719226.png" alt="image-20201006141719226" style="zoom: 67%;" />

##### 6.2 Changing shade styles

Instead of Bezier curves, we can change the shade area into lines:

```shell
#changing shade style to line
python -m jcvi.graphics.karyotype seqids layout --shadestyle=line
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201006142050441.png" alt="image-20201006142050441" style="zoom: 67%;" />

##### 6.3 Adding more genomes

We can also add in as many genomes as we want. However, do note that if we want to compare between any two genomes, we would have to generate a `.simple` file between those two genomes. For example, `GenomeA.GenomeB.anchors.simple` and `GenomeB.GenomeC.anchors.simple` would allow us to compare between Genomes A and B as well as Genomes B and C, but not Genomes A and C.

For this example, I added in another sequence containing genomic islands from *Burkholderia thailandensis* E264 Chromosome 1. Similarly, I start with two input files: `e264chr1.faa` and  `e264chr1.gff` and I generate the corresponding `e264chr1.bed` and `e264chr1.pep` files.

Lets compare pinkchr1 vs. purplechr1 vs. e264chr1:

```shell
#since we already have pinkchr1 vs. purplechr1, lets generate a pairwise synteny between purplechr1 and e264chr1
#you can submit a pbs script for this, by editing the script from Section 4.1
python -m jcvi.compara.catalog ortholog purplechr1 e264chr1 --no_strip_names --dbtype prot --cscore=0.99

#generating .simple file from .anchors file
python -m jcvi.compara.synteny screen --minspan=30 --simple purplechr1.e264chr1.anchors purplechr1.e264chr1.anchors.new
```

##### 6.3.1 Updating `seqids` file 

Next, we would need to update our `seqids` file.

```shell
#editing seqids file
nano seqids
```

 `e264chr1` has 6 genomic islands, named GI1-GI6. Save and exit:

```tex
GI1,GI2,GI3,GI4,GI5,GI6,GI7,GI8
GI1,GI2,GI3,GI4,GI5,GI6,GI7,GI8,GI9
GI1,GI2,GI3,GI4,GI5,GI6
```

##### 6.3.2 Updating `layout` file

Let's update the `layout` file next:

```shell
#editing layout file
nano layout
```

We can change the plot coordinates and rotation in the file according to how we want our final figure to be. For now, let us proceed with this:

```shell
# y, xstart, xend, rotation, color, label, va,  bed
 .7,     .2,    .9,       0,      , pinkchr1, top, pinkchr1.bed
 .5,     .2,    .9,       0,      , purplechr1, top, purplechr1.bed
 .3,     .2,    .7,       0,      , e264chr1, bottom, e264chr1.bed
# edges
e, 0, 1, pinkchr1.purplechr1.anchors.simple
e, 1, 2, purplechr1.e264chr1.anchors.simple

```

> ***Important***:
>
> Make sure there is no space (` `)  at the start of each line below `# edges`.

##### 6.3.3 Plotting figure

Now we have our input files ready, we can plot.

```shell
#plotting
python -m jcvi.graphics.karyotype seqids layout
```

This generates another  `karyotype.pdf` that we can download and visualize:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007122200941.png" alt="image-20201007122200941" style="zoom: 67%;" />

##### 6.3.4 Highlighting a specific block across multiple alignments

Looking at the plot, we notice that pinkGI3 ≈ purpleGI6' ≈ e264GI5'. Let us highlight this specific block. Remember that jcvi compares genomes in pairs, so we would have to download both `pinkchr1.purplechr1.anchors.simple` and `purplechr1.e264chr1.anchors.simple` and update the corresponding blocks with a specific colour that we want.

`pinkchr1.purplechr1.anchors.simple`:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007122921562.png" alt="image-20201007122921562" style="zoom:80%;" />

`purplechr1.e264chr1.anchors.simple`:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007122954848.png" alt="image-20201007122954848" style="zoom:80%;" />

Save and import both `.simple` files back into FileZilla. Lets plot:

```shell
#plotting
python -m jcvi.graphics.karyotype seqids layout
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007123211419.png" alt="image-20201007123211419" style="zoom: 67%;" />

##### 6.3.5 Rotating alignments

We can explore this function too, which gives us some pretty fancy plots. All we need to do is edit the `layout` file and replot the figure: 

```shell
# y, xstart, xend, rotation, color, label, va,  bed
 .5,   .025,  .625,      60,      , pinkchr1, top, pinkchr1.bed
 .2,     .2,    .8,       0,      , purplechr1, top, purplechr1.bed
 .5,   .375,  .975,     -60,      , e264chr1, top, e264chr1.bed
# edges
e, 0, 1, pinkchr1.purplechr1.anchors.simple
e, 1, 2, purplechr1.e264chr1.anchors.simple
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007123838027.png" alt="image-20201007123838027" style="zoom: 67%;" />

However, this figure only shows synteny between pinkchr1 and purplechr1, purplechr1 and e264chr1. Let us add in pinkchr1 and e264chr1:

```shell
#running pairwise synteny between pinkchr1 and e264chr1
python -m jcvi.compara.catalog ortholog pinkchr1 e264chr1 --no_strip_names --dbtype prot --cscore=0.99

#generating corresponding .simple file
python -m jcvi.compara.synteny screen --minspan=30 --simple pinkchr1.e264chr1.anchors pinkchr1.e264chr1.anchors.new
```

Let us update the `.simple` file to highlight our block of interest. Remember to save changes before uploading the file into Filezilla:

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007125252494.png" alt="image-20201007125252494" style="zoom:80%;" />

And update the `layout` file accordingly:

```shell
# y, xstart, xend, rotation, color, label, va,  bed
 .5,   .025,  .625,      60,      , pinkchr1, top, pinkchr1.bed
 .2,     .2,    .8,       0,      , purplechr1, top, purplechr1.bed
 .5,   .375,  .975,     -60,      , e264chr1, top, e264chr1.bed
# edges
e, 0, 1, pinkchr1.purplechr1.anchors.simple
e, 1, 2, purplechr1.e264chr1.anchors.simple
e, 0, 2, pinkchr1.e264chr1.anchors.simple
```

Finally, let's plot:

```shell
#plotting
python -m jcvi.graphics.karyotype seqids layout
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201007125528907.png" alt="image-20201007125528907" style="zoom: 67%;" />

> *'it's quite a feat to choose dataset until can draw this HAHAHA', SK 2020*

### 7. Microsynteny visualization

It is also possible for us to zoom in and visualize pairwise synteny at the gene-level. Microsynteny shows the matching regions along aligned gene models but first we need to compute the layout for gene-level matchings:

```shell
#computing layout
python -m jcvi.compara.synteny mcscan pinkchr1.bed pinkchr1.purplechr1.lifted.anchors --iter=1 -o pinkchr1.purplechr1.il.blocks
```

> **Note**:
>
> `--iter=1` means we are extracting *one* best region matching each pinkchr1 region. If you look at the resulting `pinkchr1.purplechr1.il.blocks` file, it contains pinkchr1 as the first column and purplechr1 as the second column. If the option `--iter` is set to 2, there will be 2 purplechr1 regions, and so on. Specifically, this will be useful to plot regions resulted from genome duplications. Example of `il.blocks` file below:
>
> <img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201006171226496.png" alt="image-20201006171226496" style="zoom:80%;" />

##### 7.1 Selecting plot regions

The `pinkchr1.purplechr1.il.blocks` contains lots of local regions. We can choose any arbitrary region to visualize:

```shell
#selecting region to include in plot
head -25 pinkchr1.purplechr1.il.blocks > blocks
```

> Still figuring out context of head, -25. Plot first 25 genes starting from the front?

##### 7.2 Generating `layout` file

Similarly, we will need a `layout` file just like in macrosynteny plots. Create a `blocks.layout` file:

```shell
#creating blocks.layout file
nano blocks.layout
```

 Copy the following into the terminal. (ha = horizontal alignment, va = vertical alignment). Save and exit:

```tex
# x,   y, rotation,   ha,     va,   color, ratio,            label
0.5, 0.6,        0, left, center,       m,     1,         pinkchr1
0.5, 0.4,        0, left, center, #fc8d62,     1,       purplechr1
# edges
e, 0, 1
```

> ***Important***:
>
> Make sure there is no space (` `)  at the start of each line below `# edges`.

##### 7.3 Generating local synteny plot

```shell
#concatenating .bed files
cat pinkchr1.bed purplechr1.bed > pinkchr1_purplechr1.bed
```

Now, we are ready to plot:

```shell
#plotting
python -m jcvi.graphics.synteny blocks pinkchr1_purplechr1.bed blocks.layout
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008173125665.png" alt="image-20201008173125665" style="zoom:67%;" />

##### 7.4 Changing plot styles

Similar to macrosynteny plots, we can have alternative shade styles:

```shell
#changing shade style to line
python -m jcvi.graphics.synteny blocks pinkchr1_purplechr1.bed blocks.layout --shadestyle=line
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008174337388.png" alt="image-20201008174337388" style="zoom:67%;" />

```shell
#alternative arrow glyph style
python -m jcvi.graphics.synteny blocks pinkchr1_purplechr1.bed blocks.layout --glyphstyle=arrow
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008174324252.png" alt="image-20201008174324252" style="zoom:67%;" />

```shell
#adding gene labels
python -m jcvi.graphics.synteny blocks pinkchr1_purplechr1.bed blocks.layout --genelabelsize 4
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008174457477.png" alt="image-20201008174457477" style="zoom:67%;" />

```shell
#adding scalebar
python -m jcvi.graphics.synteny blocks pinkchr1_purplechr1.bed blocks.layout --scalebar
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008174821456.png" alt="image-20201008174821456" style="zoom: 80%;" />

### 8. Microsynteny getting fancy

Similar to macrosynteny plots, we can also include more that two matching regions. Let's make a `blocks` file with the three genomes, using pinkchr1 as the reference, and aligning purplechr1 and e264chr1 separately to pinkchr1.

We already did pink vs. purple, so all that is left is to do the same for e264:

```shell
#computing layouts
python -m jcvi.compara.synteny mcscan pinkchr1.bed pinkchr1.e264chr1.lifted.anchors --iter=1 -o pinkchr1.e264chr1.il.blocks
```

##### 8.1 Combining `blocks` and selecting plot regions

Next, we can use the following command to combine our `blocks` files:

```shell
#combining pink vs purple and pink vs e264 blocks files
python -m jcvi.formats.base join pinkchr1.purplechr1.il.blocks pinkchr1.e264chr1.il.blocks --noheader | cut -f1,2,4,6 > pinkchr1.blocks
```

We can download `pinkchr1.blocks` and visualize it, which looks something like this (notice 3 columns):

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008183232887.png" alt="image-20201008183232887" style="zoom:80%;" />

Next, let's select our region of interest to plot:

```shell
#selecting region to include in plot
head -25 pinkchr1.blocks > blocks2
```

##### 8.2 Generating `layout` file

Let's create a new layout file called `blocks2.layout`.

```shell
#creating blocks2.layout file
nano blocks2.layout
```

Copy the following into the terminal. Save and exit:

```tex
# x,   y, rotation,     ha,     va,   color, ratio,            label
0.5, 0.6,        0, center,    top,        ,     1,         pinkchr1
0.3, 0.4,        0, center, bottom,        ,   0.5,       purplechr1
0.8, 0.4,        0, center, bottom,        ,   0.5,         e264chr1
# edges
e, 0, 1
e, 0, 2
```

> ***Important***:
>
> Make sure there is no space (` `)  at the start of each line below `# edges`.

##### 8.3 Generating local synteny plot with multiple regions

```shell
#concatenating .bed files
cat pinkchr1.bed purplechr1.bed e264chr1.bed > pinkchr1_purplechr1_e264chr1.bed
```

Now, we are ready to plot:

```shell
#plotting
python -m jcvi.graphics.synteny blocks2 pinkchr1_purplechr1_e264chr1.bed blocks2.layout --glyphstyle=arrow 
```

<img src="C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20201008184324657.png" alt="image-20201008184324657" style="zoom:67%;" />

------

