# MCScan (Python-version) Documentation

by Mark Chan

14 September 2020

MCScan (Python-version) [github](https://github.com/tanghaibao/jcvi/wiki/MCscan-(Python-version))

------

### 1. Installations

##### 1.1 Installing LAST

Obtain the latest version of LAST [here](http://last.cbrc.jp/). 

Right click last-1080.zip > copy link address.

```shell
# Go to home/users/ntu/mark0026
cd ~

# Install LAST
wget http://last.cbrc.jp/last-1080.zip
unzip last-1080.zip
cd last-1080.zip
make
```

##### 1.2 Installing JCVI utility libraries

JCVI is a collection of Python libraries to parse bioinformatics files, or perform computation related to assembly, annotation, and comparative genomics.

```shell
cd ~
pip install jcvi
```

### 2. Importing data

##### 2.1 Create a working directory

```shell
# Go to desired directory
cd scratch

# Create new directory 'mcscanpy' for mcscan-python version
mkdir mcscanpy
```

##### 2.2 Upload files using FileZilla

```shell
# Create test folder within mcscanpy directory
mkdir test
```

Drag and drop the following files into the `mcscanpy/test` directory:

> AWCtg1.gff
>
> AWCtg1.faa
>
> E264Ctg1.gff
>
> E264Ctg1.faa

Check for successful upload of files:

```shell
cd test
ls
```

### 3. Converting GFF files to BED format

[Galaxy](https://usegalaxy.org/) has this function, which allows you to view both files nicely in a tabular format. Alternatively, we can convert the files locally as well (only takes a few seconds).

```shell
# Converting both Ctg1,2.gff to Ctg1,2.bed
$ python -m jcvi.formats.gff bed --type=CDS --key=ID AWCtg1.gff -o AWCtg1.bed
$ python -m jcvi.formats.gff bed --type=CDS --key=ID E264Ctg1.gff -o E264Ctg1.bed

# Variables
# --type: Feature type (third column of gff file; in this case its CDS. Alternatively you can input mRNA for transcriptomics analysis)
# --key: Prefix of column 9 'Group' (in this case its ID)
# Ctg1.gff is my input file in gff format
# -o is output
# Ctg1.bed is my output file name in bed format
```

> Example `gff` file in a tabular format. *Note --type=CDS and --key=ID*:

![image-20200918132343486](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200918132343486.png)

> During the format conversion you should see something like this:
>
> ![image-20200918152447776](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200918152447776.png)

```shell
# Check for successful bed file format conversion
ls
```

### 4. Reformatting amino acid `fasta` files

```shell
$ python -m jcvi.formats.fasta format AWCtg1.faa AWCtg1.cds
$ python -m jcvi.formats.fasta format E264Ctg1.faa E264Ctg1.cds

# This generates two new outputs
# AWCtg1.cds
# E264Ctg1.cds
```

Now we have a clean set of files to proceed. We would only need `.bed` and `.cds` files.

Created a new directory synteny and move `.bed` and `.cds` files inside:

```shell
mkdir synteny
mv *.bed *.cds synteny
```

### 5. Pairwise synteny search

```shell
#enter synteny directory
cd synteny

#run pairwise synteny search
python -m jcvi.compara.catalog ortholog AWCtg1 E264Ctg1 --no_strip_names --dbtype prot
```

![image-20200921183826101](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200921183826101.png)

Observed above error in lastal, noted that during installation of lastal (make) didnt work. have to install using make CXXFLAGS=-O3

> ![image-20200921184230535](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200921184230535.png)

##### Updating of PATH dir

```shell
#check existing path
echo $PATH

#update path to include last
export PATH=$PATH:/home/users/ntu/mark0026/last-1080/src

##export function only works for current login session
##may need to check and update path again if lastal command not found for future use
```

> #old path
>
> /home/users/ntu/mark0026/miniconda3/bin:/home/users/ntu/mark0026/miniconda3/condabin:/app/pbs/bin:/app/CD-adapco/14.04.013/STAR-View+14.04.013/bin:/app/CD-adapco/14.04.013/STAR-CCM+14.04.013/star/bin:/app/pbs/bin:/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/opt/ibutils/bin:/sbin:/usr/sbin:/opt/pbs/bin:/home/users/ntu/mark0026/edirect:/home/users/ntu/mark0026/edirect:/home/users/ntu/mark0026/bin
>
> #updated path 
>
>
> /home/users/ntu/mark0026/miniconda3/bin:/home/users/ntu/mark0026/miniconda3/condabin:/app/pbs/bin:/app/CD-adapco/14.04.013/STAR-View+14.04.013/bin:/app/CD-adapco/14.04.013/STAR-CCM+14.04.013/star/bin:/app/pbs/bin:/usr/lib64/qt-3.3/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/opt/ibutils/bin:/sbin:/usr/sbin:/opt/pbs/bin:/home/users/ntu/mark0026/edirect:/home/users/ntu/mark0026/edirect:/home/users/ntu/mark0026/bin:/home/users/ntu/mark0026/last-1080/src



To convert cds to pep : `cp xx.cds xx.pep`

```shell
#error:
lastal: I was installed here with multi-threading disabled

#need to remove thread (-P < >)
#solution:
lastal -u 0 -i3G -f BlastTab E264Ctg1cheat E264Ctg1.pep >./E264Ctg1.E264Ctg1cheat.last
```



Next, we need to install Texlive. But suspected that conda version of Texlive is outdated and buggy when run on mcscanpy (can try, just in case)

```shell
#installing TeXlive core
conda install -c conda-forge texlive-core


##suspected conda version is outdated/contains errors when run on mcscanpy
```



If it doesn't work, we need to download Texlive manually:

```shell
#download texlive manually
wget https://download.nus.edu.sg/mirror/ctan/systems/texlive/tlnet/install-tl.zip

unzip install-tl.zip

./intall-tl

##suspected error of default isntallation to root dir in local. hence need to change and specific dir to writable dir (not scratch due to concerns of auto-delete after 1 month of inactivity )
#option : D to change dir
#option : 1 to set dir
## to set dir to
/home/users/ntu/mark0026/texlive/2020

#enter
#R to return


#updating PATH to include texlive
export PATH=$PATH:/home/users/ntu/mark0026/texlive/2020/bin/x86_64-linux

##export function only works for current login session

#to check if texlive was successfully installed
tex --help

#not sure if conda texlive has conflicts with manually downloaded one
#suspected conda texlive has conflicts with manually downloaded one
#can delete conda texlive if previously installed
conda remove texlive-core
```



![image-20200921192335973](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200921192335973.png)



After all that is done, we can try to run pairwise synteny search again:

```shell
#current command for pairwise synteny

python -m jcvi.compara.catalog ortholog AWCtg1 E264Ctg1 --no_strip_names --dbtype prot

#test with 'cheat' dataset consisting of duplicates of exact same protein /genome
python -m jcvi.compara.catalog ortholog AWCtg1 AWCtg1cheat --no_strip_names --dbtype prot

#after running above, there is an error of lastal multi-thread
#checked that default lastal script invoked from jcvi include flag -P 24 (e.g. threading of 24)

#Therefore, launch manual lastal:
lastal -u 0 -i3G -f BlastTab AWCtg1cheat AWCtg1.pep >./AWCtg1.AWCtg1cheat.last


```



Tried running pairwise synteny, got the following error:

![image-20200922120513242](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200922120513242.png)

```shell
#Followed solution from
##https://stackoverflow.com/questions/50564999/lib64-libc-so-6-version-glibc-2-14-not-found-why-am-i-getting-this-error

1. mkdir ~/glibc214
2. cd ~/glibc214
3. wget http://ftp.gnu.org/gnu/glibc/glibc-2.14.tar.gz
4. tar zxvf glibc-2.14.tar.gz
5. cd glibc-2.14
6. mkdir build
7. cd build
8. ../configure --prefix=/opt/glibc-2.14
9. make -j4 #for screensaver
10. make install #for screensaver
11. export LD_LIBRARY_PATH=/opt/glibc-2.14/lib #for current login session. will have to export this to $PATH every time re-logging in

#updated solution guide for step 8 onwards
8. ../configure --prefix=/home/users/ntu/mark0026/glibc214/glibc-2.14-install
9. make -j4
10. make install
11. export LD_LIBRARY_PATH=/home/users/ntu/mark0026/glibc214/glibc-2.14-install/lib
 echo $LD_LIBRARY_PATH
 
 
```

After all that's done, seems like mcscanpy is working. But problems with generating `.pdf` document containing the plots was still an issue.

![image-20200922134308471](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200922134308471.png)

```shell
#checking output files
ls AWCtg1.E264Ctg1.*
```

> You should see the following output

![image-20200922134409420](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200922134409420.png)

The `.last` file is raw LAST output, `.last.filtered` is filtered LAST output, `.anchors` is the seed synteny blocks (high quality), `.lifted.anchords` 



> Had problems with `.pdf` output and visualizing the plots.

![image-20200922180004950](C:\Users\markc\AppData\Roaming\Typora\typora-user-images\image-20200922180004950.png)