# SNP-heritability estimation for Biobank
Now we demonstrate a walkthrough on how to estimate SNP-heritability for GWAS in Biobank scale (e.g. UK Biobank) starting from genotype and phenotype data. 

## Data

Prepare the following data:

- `all_bfile`: Genotype for the **unrelated** individuals in UK Biobank after quality control. This contains genome-wide data in one file.
- `pheno_file`: the phenotype file.
- `covar_file`: the covariates which will be used for association studies.

!!! note "Data requirement for GRE"
    GRE is designed for estimating SNP-heritability at Biobank scale. Only genotyped SNPs should be included. Also make sure that **number of individuals > number of snps on each chromosome**. In our [simulation studies](https://rdcu.be/bMhni), we find when the #individuals=337k, genome-wide #SNPs=600k, largest chromosome #SNPs=60k, GRE works great.

## Variables in the script

Here is a summary of variables we will use in the following script and also the file structure we are going to create.


!!! note "Note on the scripts"
    Note that there are some parameters about the memory and time shown in the scripts below. These parameters are ones we used to perform analyses with data #individuals=337k, #genome-wide SNPs=600k. You might need to tune them according to the size of the data. But be sure to note the consistency. If you have questions, feel free to contact us.

    Also, since h2-GRE is designed to analyze biobank-scale data, we assume you have a computing cluster in your organization. And note that the commands for submitting jobs (we use `qsub` command) to the computing cluster may vary between different sysmtems.



## Association summary statistics
To use GRE, be sure to perform the association studies with **the exactly same genotype** you use to compute the LD. We leave GRE with reference panel LD for future work.

We compute the OLS sumstats using plink.

First create a bash script called `assoc.sh`.
```bash
#!/bin/sh
#$ -l h_data=20G,h_rt=24:00:00,highp
#$ -cwd
#$ -j y
#$ -o ./job_out
#$ -t 1-22:1

source /u/local/Modules/default/init/modules.sh
module load plink

chr_i=$SGE_TASK_ID
num_cols=$(head -1 $covar_file | awk '{print NF}')
num_cols=$((num_cols-2))

mkdir -p ./assoc

plink --bfile ${all_bfile} \
  --chr $chr_i \
  --linear hide-covar \
  --ci 0.95 \
  --pheno ${pheno_file} \
  --allow-no-sex \
  --covar ${covar_file} \
  --covar-number 1-${num_cols} \
  --out ./assoc/chr${chr_i}
```
```bash
qsub assoc.sh
```

You should get 22 association file after this step. Each contains the OLS sumstats for SNPs in 22 chromosome.

## Estimating  using GRE

### Divide the genotype into 22 parts by chromosome using plink
```bash
# now in root directory
mkdir genotype

for chr_i in $(seq 1 22)
do
    plink \
    --bfile ${all_bfile}
    --chr ${chr_i}
    --make-bed
    --out genotype/chr${chr_i}
done
```

### Download GRE
```bash
git clone https://github.com/bogdanlab/h2-GRE.git
```



### Create 22 directory to store intermediate files used by GRE

```bash
# now in root directory
mkdir ld
for chr_i in $(seq 1 22)
do
    mkdir ld/chr${chr_i}
done
```

## Compute the mean and standard deviation for each genotype file

`mean_std.sh`
```bash
#!/bin/sh
#$ -cwd
#$ -j y
#$ -l h_data=8G,h_rt=1:00:00
#$ -o ./job_out
#$ -t 1-22

chr_i=$SGE_TASK_ID
bfile=./genotype/chr${chr_i}
ld_dir=./ld/chr${chr_i}
chunk_size=500
gre_path=./h2-GRE/gre/gre.py
python ${gre_path} mean-std --bfile=${bfile} --chunk-size=${chunk_size} --ld-dir=${ld_dir}
```

```bash
qsub mean_std.sh
```

This step will produce a file containing the mean and standard deviation for each genotype file.

In the following few steps, we compute the LD matrix efficiently. We achieve this by seperating the big task, e.g., computing the 60k x 60k SNP covariance matrix into computing 3600 1k x 1k SNP covariance matrix.

### Cutting the task of computing LD matrix into several tasks

```bash
for chr_i in $(seq 1 22)
do
    python ${gre_path} cut-ld --bfile ./genotype/chr${chr_i} --chunk-size 1000 --ld-dir ./ld/chr${chr_i}
    python ${gre_path} check-ld ./ld/chr${chr_i}
done
```

This will output two files `part.info`, `part.missing` in each directory `./ld/chr${chr_i}`, which specifies the sub tasks needed to compute the LD.

### Computing each part of covariance matrix
First create two scripts, `calc_ld.sh`, `run_calc_ld.sh`.

`calc_ld.sh`

```bash
#!/bin/sh
#$ -cwd
#$ -l h_data=16G,h_rt=0:40:00
#$ -j y
#$ -o ./job_out

chr_i=$1
batch_size=$2

bfile=./genotype/chr${chr_i}
ld_dir=./ld/${chr_i}
gre_path=./h2-GRE/gre/gre.py
for i in $(seq 1 ${batch_size})
do
    line=$((SGE_TASK_ID + i - 1))
    # part.missing contains undone tasks
    task_id=$(awk 'NR == n' n=$line ${ld_dir}/part.missing)
    python ${gre_path} calc-ld --bfile=${bfile} --part-i=${task_id} --ld-dir=${ld_dir}
done
```

`run_calc_ld.sh`

```bash
batch_size=25
for chr_i in $(seq 1 22)
do
    ld_dir=./ld/${chr_i}
    part_num=$(wc -l < ${ld_dir}/part.missing)
    if [ ${part_num} -ne 0 ]; then
        qsub -t 1-${part_num}:${batch_size} calc_ld.sh ${chr_i} ${batch_size}
    fi
done
done
```
Then run the `run_calc_ld.sh` to submit the jobs.
```bash
bash run_calc_ld.sh
```
### check if jobs are done and resubmit the jobs if necessary
Some of jobs might not finish due to slow nodes on the cluster. We update the `part.missing` list first.
```bash
for chr_i in $(seq 1 22)
do
    python ${gre_path} check-ld --ld-dir ./ld/chr${chr_i}
done
```
And calculate the ld for each part by `bash run_calc_ld.sh`. Repeat this until you see no tasks being submitted.

### Merging parts of the covariance matrix
Now we are close to finishing it! We will merge each parts of LD and compute the pseudo-inverse of LD.


`merge_ld.sh`
```bash
#!/bin/sh
#$ -cwd
#$ -j y
#$ -o ./job_out
#$ -m a

bfile=$1
ld_dir=$2
gre_path=./h2-GRE/gre/gre.py
python ${gre_path} merge-ld --bfile=${bfile} --ld-dir=${ld_dir}
```

```bash
for chr_i in $(seq 1 22)
do
    ld_dir=./ld/chr${chr_i}
    bfile=./genotype/chr${chr_i}
    if [ $chr_i -le 6 ]
    then
        qsub -l h_data=60G,h_rt=4:00:00,highp merge_ld.sh ${bfile} ${ld_dir}
    elif [ $chr_i -ge 7 -a $chr_i -le 12 ]
    then
        qsub -l h_data=40G,h_rt=3:00:00,highp merge_ld.sh ${bfile} ${ld_dir}
    elif [ $chr_i -ge 13 -a $chr_i -le 20 ]
    then
        qsub -l h_data=24G,h_rt=2:00:00,highp merge_ld.sh ${bfile} ${ld_dir}
    else
        qsub -l h_data=10G,h_rt=1:00:00,highp merge_ld.sh ${bfile} ${ld_dir}
    fi
done
```

Now we have all the ingredients needed for GRE and we are ready to get the SNP-heritability.

### Estimating SNP-heritability
Converting the sumstats in plink format into LDSC format.
```bash
for chr_i in $(seq 1 22)
do
    python ./h2-GRE/utils/utils.py sumstats-plink2ldsc --plink-sumstats ./assoc/chr${chr_i}.assoc.linear --legend ./genotype/chr${chr_i}.bim --out ./assoc/chr${chr_i}.ldsc.txt
done
```

Estimate SNP-heritability

`estimate.sh`
```bash
#!/bin/sh
#$ -cwd
#$ -l h_data=24G,h_rt=2:00:00,highp
#$ -j y
#$ -o ./job_out
#$ -t 1-22:1

chr_i=${SGE_TASK_ID}
legend=./genotype/chr${chr_i}.bim
ld_dir=./ld/chr${chr_i}
sumstats=./assoc/chr${chr_i}.ldsc.txt
python gre.py estimate --legend=${legend} --ld-dir=${ld_dir} --sumstats=${sumstats} > ./estimate/chr${chr_i}.txt
```
```bash
qsub estimate.sh
```
`./estimate/chr${chr_i}.txt` contains the estimate and standard deviation of GRE for each chromosome. Adding these up provides you the genome-wide SNP-heritability!



