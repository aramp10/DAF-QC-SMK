# DAF-seq processing and QC pipeline

[![Actions Status](https://github.com/StephanieBohaczuk/DAF-QC-SMK/workflows/CI/badge.svg)](https://github.com/StephanieBohaczuk/DAF-QC-SMK/actions)


## Install

Please start by installing [pixi](https://pixi.sh/latest/) which handles the environment of this Snakemake workflow.

You can then install the `pixi` environment by cloning this repository and running:

```bash
pixi install
```

## Usage

`pixi` handles the execution of the Snakemake workflows. You can run it with either of the following:


```
pixi shell
snakemake --configfile config/config.yaml
```


```bash
pixi run snakemake ...
```

And if you want to run this Snakemake from another directory you can do so with:

```bash
pixi run --manifest-path /path/to/snakemake/pixi.toml snakemake ...
```

where you update `/path/to/snakemake/pixi.toml` to the path of the `pixi.toml` you cloned.

And in place of `...` use all the normal Snakemake arguments for your workflow.


To run on a SLURM cluster, you will need to create and specify a profile. See Snakemake's documentation for instructions. UW Hyak users can use the provided profile in `profiles/slurm-executor/`. You can then run with:

```bash
pixi run snakemake --configfile config/config.yaml --profile profiles/slurm-executor
```

## Running the test case

A dataset, config file, and fasta sequence for human chr 8 (hg38) is provided for testing and can be downloaded and run with the following from the DAF-QC-SMK directory:

```bash
pixi run test
```
To better assist in troubleshooting, please make sure to run the test case if you are contacting us regarding errors.

## Inputs
`config.tbl` Table that includes sample name, bam/FASTA path, and targeted regions. For compatibility with pbmarkdup, PacBio bam file inputs should be either unaligned or filtered for primary reads only if a consensus will be generated.
See `config/config.tbl` for a template.
```
sample	file	regs
test_sample	/my/bam/path/file.bam	chr4:3073138-3075853,chr3:179228176-179236561
``` 

`config.yaml` Config file that includes paths to sample table and reference genome and specifies custom parameters.
See `config/config.yaml` for a template.
```
ref: /my/ref/path/genome.fa # path to a reference fasta
manifest: config/config.tbl # table with samples to process
platform: pacbio # sequencing platform, either 'pacbio' or 'ont'. Default is pacbio

# Optional (both platforms)
chimera_cutoff: 0.9 # minimum fraction of (C->T|G->A)/(C->T+G->A) required for a top or bottom strand designation (i.e. non-chimeric)
min_deamination_count: 50 # minimum number of deaminations for top/bottom strand designation
end_tolerance: 30 # +/- end length tolerance (in bp) for classifying full-length reads
decorated_samplesize: 5000 # Approximate number of reads to output as decorated reads bam for visualization
benchmark: False # Whether to save benchmarking for run time and memory usage. Default is False.

# PacBio-specific options. Input should always be bam files.
dup_end_length: 0 # length to consider for deduplication. 0 will consider the whole read.
dup_min_id_perc: 99.2 # minimum identity percentage required to mark duplicates
consensus: True # whether to generate a consensus sequence for each read. Default is true
consensus_min_reads: 3 # minimum number of reads required to generate a consensus sequence

# ONT-specific options
is_fastq: False # for ONT files only, specifies whether the input files are fastq, otherwise they are assumed to be bam files
```

## Outputs
### BAM files

`results/{sample_name}/align/{sample_name}.mapped.reads.bam` Aligned bam containing all primary, supplementary, and unaligned reads with PCR duplicates marked (du and ds tags)

`results/{sample_name}/align/{sample_name}.decorated.bam` bam containing full-length reads designated top/bottom with C->T as Y(top strand) and G->A as R (bottom strand). Strand designation is stored in the `st` tag. If `decorated_samplesize` is specified in the config file, this will contain a randomly selected sample of full length, top/bottom strand reads.

`results/{sample_name}/align/{sample_name}.mapped.consensus.bam` bam containing MSA consensus of full length, top/bottom reads with a minimum number of reads specified by dups_required (default:3). Consensus read names are `pbmarkdup` `du` tags with "_consensus" appended. The `du` tag is also included. The `zm` tag is arbitrarily set to that of the `du` tag name. The `dc` tag indicates the number of reads that were used to construct the consensus. For visualization, the `YC` tag colors consensus reads pink. For `consensus_min_reads` that include values <3, the consensus for groups with 2 reads will be one of the reads in the duplicate group, typically the pbmarkdup representative (`du` tag read name), and the consensus for groups with 1 read will be that read. Note that `consensus_min_reads` <3 is not recommended for most applications as it may lead to an over-representation of poorly sequenced reads.


### Data files
`results/{sample_name}/qc/reads/{sample_name}.deduplication_metrics.tbl.gz` For each region, contains a comma-separated list of group names designated by `pbmarkdup` and number of reads in each duplicate group. Note that only full-length, top/bottom strand reads are considered, so numbers may differ from the distribution of `ds` tags from pbmarkdup.
```
chrom   start   end     du_tags values
chr4    3073138 3075853 m21034_250307_213248/112462026/ccs,... 100,50,42,....
```

For the file types below, detailed and summary metrics are provided. Detailed metrics provide all the information used in filtering and strand designation. Summary metrics are provided for plot generation.

`results/{sample_name}/qc/reads/{sample_name}.detailed_targeting_metrics.tbl.gz` For each region, contains read names designated as full-length, non-full length, unaligned, and supplementary/secondary alignment.
```
chrom   start   end     full_length_reads       non_full_length_reads   non_primary_reads
chr4    3073138 3075853 m21034_250307_213248/84281987/ccs,...   m21034_250307_213248/46338585/ccs,...   m21034_250307_213248/115871607/ccs,...
```

`results/{sample_name}/qc/{sample_name}.summary_targeting_metrics.tbl` For each region, contains the number of reads in each category
```
chrom   start   end     #_full_length_reads     #_non_full_length_reads #_non_primary_reads     total_fibers in bam(primary+unmapped)
chr4    3073138 3075853 20837   1746    261     22736
```

`results/{sample_name}/qc/{type}/{sample_name}.detailed_seq_metrics.{type}.tbl.gz` Contains strand by strand overall deamination, 2bp deamination sequence context, and mutation metrics for full length reads (type=reads) or consensus sequences (type=consensus). Note that "OC" indicates that the base is not in a 2bp context (i.e. at the end of the read or next to an indel). Context is determined from the reference sequence to avoid ambiguity from deaminated bases.
```
read_name       chr     reg_st  reg_end strand_st       strand_end      length  strand  duplicate       mutation_count  deamination_positions   AC_count    AC_deam AC_deam_rate    CC_count        CC_deam    CC_deam_rate    GC_count        GC_deam GC_deam_rate    TC_count        TC_deam TC_deam_rate    OC_count        OC_deam OC_deam_rate    total_count      total_deam     all_deam_rate   mutation_rate
m21034_250307_213248/84281987/ccs       chr4    3073138 3075853 3073130 3075848 2731    CT      None    3.0     176,193,...      105   16    0.1523809523809524      421   87    0.20665083135391923     322   53    0.16459627329192547     166   54       0.3253012048192771      2     0     0.0     1016	210   0.20669291338582677     0.0010984987184181618
```

`results/{sample_name}/qc/{type}/{sample_name}.summary_seq_metrics.{type}.tbl.gz` Contains proportions of deaminations by region, strand type, and 2 bp context for full length reads or consensus sequences. "OC" column indicates 
```
chrom   reg_start       reg_end strand  count   mutation_rate   all_deam_rate   AC_deam_rate    CC_deam_rate    GC_deam_rate    TC_deam_rate    OC_deam_rate
chr4    3073138 3075853 CT      15855   0.0010984987184181618,0.0007334066740007334,... 0.30912659470068693,0.2992125984251969,... 0.23076923076923078,0.16831683168316833... 0.2327790973871734,0.34523809523809523    0.36335403726708076,0.21846153846153846,... 0.45180722891566266,0.47305389221556887,... 0,0.25,0,...
```

### Plots
The easiest way to visualize plots is through the HTML dashboard in `results/{sample_name}/qc/{sample_name}.dashboard.html`. Plots are also avaiable as individual pdf files in `results/{sample_name}/qc/{type}/plots/` where type is either `reads` or `consensus`. Note that the plots are embedded in the HTML, so it is not necessary to include individual PDF files when copying to your local machine or sharing results.

Plots include targeting efficiency, deamination rate, strand calling, enzyme bias, mutation rate, and deduplication (PacBio only) at the read level, and deamination rate, strand calling, enzyme bias, and mutation rate at the consensus level (if consensus is generated).


## Acknowledgements

Thank you to Mitchell Vollger for providing the template for this Snakemake workflow and pixi.toml, and for some of the common functions in [common.smk](workflow/rules/common.smk)

# DAF-QC-SMK Pipeline Setup on BU SCC

This guide explains how to run the **DAF-QC-SMK** pipeline on the BU SCC with a local build of **pyabpoa** for compatibility.
---

## Step 1: Build `pyabpoa` Locally

1. Navigate to your SCC project directory:
   ```bash
   cd /your/projectnb/directory
   # Example:
   cd /projectnb/ar-rcs/client/cvmar
   ```

2. Clone the `abPOA` repository and checkout version **1.5.4**:
   ```bash
   # Note: the [--recursive] flag is required as the GitHub repo has submodules (.gitmodules present in repo)
   git clone --recursive https://github.com/yangao07/abPOA.git
   cd abPOA
   git checkout v1.5.4
   ```

3. Load Python 3.10 (required by Snakemake pipeline):
   ```bash
   module load python3/3.10.12
   ```

4. Edit the **Makefile**:
   - Change line 53:
     ```text
     SIMD_FLAG = -march=native
     ```
     to:
     ```text
     SIMD_FLAG = -march=sandybridge
     ```

5. Build the wheel file:
   ```bash
   python setup.py bdist_wheel
   ```
---

## Step 2: Obtain the Pipeline and Prepare Environment

1. Navigate back to your SCC project folder:
   ```bash
   cd /projectnb/ar-rcs/client/cvmar
   ```

2. Clone the pipeline repository:
   ```bash
   git clone https://github.com/StergachisLab/DAF-QC-SMK
   ```

3. Copy the wheel file to the pipeline directory:
   ```bash
   SCC_PROJECT_DIR=/projectnb/ar-rcs/client/cvmar
   cp ${SCC_PROJECT_DIR}/abPOA/dist/pyabpoa-1.5.4-cp310-cp310-linux_x86_64.whl ${SCC_PROJECT_DIR}/DAF-QC-SMK
   ```

4. Unload Python module (no longer needed):
   ```bash
   module unload python3/3.10.12
   ```
---

## Step 3: Modify Pipeline Environment File

1. Navigate to the workflow directory:
   ```bash
   cd ${SCC_PROJECT_DIR}/DAF-QC-SMK
   ```
2. Edit `workflow/envs/python.yaml`:
   - **Remove**:
     ```yaml
     - pyabpoa==1.5.4
     ```
   - **Add** this to the end of the file. The spaces before the - characters are critical, keep them!:
     ```yaml
       - pip
       - pip:
         - /net/scc-i02/scratch/bgregor/DAF-QC-SMK/pyabpoa-1.5.4-cp310-cp310-linux_x86_64.whl
     ```
     Note: there are two spaces preceeding the `- pip` lines and four spaces preceeding the `- /net/...` line.

3. Backup files provided by Brian:
   ```bash
   mkdir ${SCC_PROJECT_DIR}/bkup
   cp /net/scc-i02/scratch/bgregor/to_pushpinder/* ${SCC_PROJECT_DIR}/bkup
   ```
---

## Step 4: Clean Previous Test Runs
```bash
cd ${SCC_PROJECT_DIR}/DAF-QC-SMK
rm -rf dafqc-test-data
```
---

## Step 5: Test the Pipeline

1. Confirm `pixi` is installed:
   ```bash
   pixi --version  # Expected: pixi 0.61.0
   ```

2. Run the test:
   ```bash
   time pixi run test
   ```
   - This will download test data, build conda environments, and execute the workflow.
   - Expected runtime: ~7 minutes.
   - Logs: `.snakemake/log/<timestamp>.snakemake.log`
---

### Result
- Pipeline should complete successfully:
  ```
  Finished job 0.
  17 of 17 steps (100%) done
  ```
## Key Details
* Ticket: [INC20792462](https://bu.service-now.com/now/nav/ui/classic/params/target/incident.do%3Fsysparm_tiny%3D9dfc3f79974e69d09a3a7be0f053afaf%26sys_id%3D0d0653ec933132d034e4b0cdfaba1083%26sysparm_record_row%3D7)