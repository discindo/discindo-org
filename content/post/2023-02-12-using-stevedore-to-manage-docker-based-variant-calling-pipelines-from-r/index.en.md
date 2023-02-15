---
title: Using {stevedore} to manage docker-based variant-calling pipelines from R
author: teo
date: '2023-02-12'
slug: using-stevedore-to-manage-docker-based-variant-calling-pipelines-from-r
categories: 
  - R
  - Bioinformatics
tags: 
  - stevedore
  - docker
  - gatk
  - deepVariant
subtitle: ''
summary: ''
authors: [teo]
lastmod: '2023-02-12T17:05:06-06:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
show_related: true
---

# Docker-based variant-calling pipelines

Encapsulating large and complex software applications or packages in `docker` images
makes sense for many reasons. Not least because of a one-line install protocol 
(i.e., `docker pull`) that takes care of the plethora of dependencies that many of these
packages have. This is especially true in bioinformatics, where a differential 
gene expression workflow or a variant-calling pipeline commonly have several dependencies
written in different languages and require installation of each before the final program
can be installed. 

Two of the most widely used software packages for calling genomic variants, 
[`gatk`](https://gatk.broadinstitute.org/hc/en-us) and [`deepVariant`](https://github.com/google/deepvariant),
encourage the use of `docker` images for installation and analysis. Building a 
variant calling pipeline using one of these tools at some point would require
running the respective `docker` image from the command-line or calling `docker` by 
sending a system command to the shell from our programming environment. 
For example, in `Python`, we might use `subprocess` and in `R` we could call `system` to
make something happen externally. 

This adds a bit of unnecessary overhead. Either we have to orchestrate `Python/R/shell` scripts together,
or we have to construct system calls, capture their output (`stdin/stderr`), and then evaluate this 
within our programming environment to handle errors and warnings or proceed. 

To do this in `R` we might use base `R`'s `system`, `system2`, or the package 
[`{sys}`](https://github.com/jeroen/sys), but would it not be much better to 
call `docker` directly, i.e., without constructing and sending a system call?

This is where [`{stevedore}`](https://github.com/richfitz/stevedore) comes in. 
`{stevedore}` is a docker client for `R` that provides an interface to the Docker API.
I started working with it recently when I was exploring the best way to push a
container to `AWS ECR` (see [here](https://discindo.org/post/deploy-an-r-script-as-an-aws-lambda-function-without-leaving-the-r-console/)).
Since then, I thought about some other cases where it would be beneficial to 
call `docker` directly, rather than through a system call, and I remembered
my variant-calling pipelines that rely on `docker`-based instances of `gatk` and
`deepVariant`.

# Run containerized `deepVariant` from `R`

Working with `{stevedore}` requires (obviously) `docker` to be installed. The first
step is always creating a `docker` client object with `docker_client`. Then, to pull the
released `deepVariant` image to our local machine, we'd run:

``` r
docker_cli <- stevedore::docker_client()
docker_cli$image$pull("google/deepvariant")

#> Pulling from google/deepvariant latest
#> Digest: sha256:83ce0d6bbe3695bcbaa348b73c48737bdbfaeaea2272b0105dd4bdfa7a804f18
#> Status: Image is up to date for google/deepvariant:latest
#> <docker_image>
#>   export()
#>   help(help_type = getOption("help_type"))
#>   history()
#>   id()
#>   inspect(reload = TRUE)
#>   labels(reload = TRUE)
#>   name()
#>   reload()
#>   remove(force = NULL, noprune = NULL)
#>   short_id()
#>   tag(repo, tag = NULL)
#>   tags(reload = TRUE)
#>   untag(repo_tag)
```

We now should have the image locally: 

``` r
docker_cli$image$list()[["repo_tags"]]

#> [[1]]
#> [1] "google/deepvariant:1.4.0"  "google/deepvariant:latest"
#> 
#> [[2]]
#> [1] "hello-world:latest"

```

and we can also start a container, here, without any arguments, so we can see
the help:

``` r
docker_cli$container$run("google/deepvariant")

#> O> Runs all 3 steps to go from input DNA reads to output VCF/gVCF files.
#> O> 
#> O> This script currently provides the most common use cases and standard models.
#> O> If you want to access more flags that are available in `make_examples`,
#> O> `call_variants`, and `postprocess_variants`, you can also call them separately
#> O> using the binaries in the Docker image.
#> O> 
#> O> For more details, see:
#> O> https://github.com/google/deepvariant/blob/r1.4/docs/deepvariant-quick-start.md
#> O> 
#> O> flags:
#> O> 
#> O> /opt/deepvariant/bin/run_deepvariant.py:
#> [long output truncated]
```

Now that we've got an instance of `deepVariant` that we can run from `R`, we need to
set up our inputs and outputs before we run the pipeline with data. This includes the 
following:

1. Set the version of `deepVariant` we want to use (the `dv_version` variable below)
2. Set the fasta reference used to map the reads. This should be `bgzip`ed and indexed (`ref` variable)
3. Set the sorted, indexed bam file with our mapped reads (`bam` variable)
4. Set the number of processors (called shards here) (`nproc` variable)

``` r
dv_version <- "1.4.0"
ref <- "reference.fa.gz"
bam <- "reads.bam"
nproc <- 4
```

The input files (and accompanying index files, etc) need to already reside in the
`input` folder, as per `deepVariant`'s [documentation](https://github.com/google/deepvariant/blob/r1.4/docs/deepvariant-quick-start.md).
This is because when we run the `docker` container, we mount our `input` folder as 
a volume that is going to be available inside the container when we run it. This
step is done by defining a character vector with two mappings of mount points. This 
is the `vol` variable below, which defines two mappings of the form `my_dir:container_dir` 
for the input and output folder. The mapping for the output sets the directory where
`deepVariant` would place the resulting vcf files at the end of the run.

``` r
img <- glue::glue('google/deepvariant:{dv_version}')

vol <- c(glue::glue("/path/to/our/input_folder:/input"),
         glue::glue("/path/to/our/output_folder:/output"))
```

After setting the image, version, and mounted volumes, we need to create a character
vector that specifies the call to `run_deepvariant` that would occur inside the container.
Think of this as the command you'd issue if you were to run the container in interactive 
mode, log into it, and then manually call `run_deepvariant` on the command line. Or,
what you would call if you had installed `deepVariant` on your machine rather than
pulled the `docker` image.

Below we use `glue` to interpolate some of the variables we defined earlier when 
passing them as arguments to the executable. For more info about all these flags,
consult the [docs](https://github.com/google/deepvariant).

``` r
cmd = c(
  glue::glue("/opt/deepvariant/bin/run_deepvariant"),
  glue::glue('--model_type=WGS'),
  glue::glue('--ref=/input/{ref}'),
  glue::glue('--reads=/input/{bam}'),
  glue::glue('--output_vcf=/output/result.vcf'),
  glue::glue('--output_gvcf=/output/result.g.vcf}'),
  glue::glue('--num_shards={nproc}'),
  glue::glue('--logging_dir=/output/logs'),
  glue::glue('--dry_run=false')
)
```

Finally, we have the three components needed to run the `deepVariant` `docker`
container from `R` in order to call variants. These are 1) the image (`img`), 
2) the volumes (`vol`), and 3) the command to execute (`cmd`). We next use `{stevedore}`
to run the image from our `R` console:

``` r
docker_cli <- stevedore::docker_client()
docker_cli$container$run(image = img, volumes = vol, cmd = cmd)

# ...
# [long output truncated]

```

And that would be it when it comes to a basic `deepVariant` run. More complex 
analyses would be done by changing the `cmd` component to add additional flags, 
for example.

# Run containerized `gatk` from `R`

The procedure is the same as for `deepVariant` with the exception of the 
mounted volumes (`vol`) and the command component (`cmd`). First, we pull the 
latest release of the `gatk` `docker` image:

``` r

docker_cli <- stevedore::docker_client()
docker_cli$image$pull("broadinstitute/gatk")

#> Pulling from broadinstitute/gatk latest
#> Digest: sha256:e7996ba655225c1cde0a1faec6a113e217758310af2cf99b00d61dae8ec6e9f2
#> Status: Image is up to date for broadinstitute/gatk:latest
#> <docker_image>
#>   export()
#>   help(help_type = getOption("help_type"))
#>   history()
#>   id()
#>   inspect(reload = TRUE)
#>   labels(reload = TRUE)
#>   name()
#>   reload()
#>   remove(force = NULL, noprune = NULL)
#>   short_id()
#>   tag(repo, tag = NULL)
#>   tags(reload = TRUE)
#>   untag(repo_tag)

```

We can test that `gatk` works by running a container:

``` r

docker_cli$container$run("broadinstitute/gatk", cmd = c("gatk", "--list"))


#> O> USAGE:  <program name> [-h]
#> O> 
#> O> Available Programs:
#> O> --------------------------------------------------------------------------------------
#> O> Base Calling:                                    Tools that process sequencing machine data, e.g. Illumina base calls, and detect sequencing level attributes, e.g. adapters
#> O>     CheckIlluminaDirectory (Picard)              Asserts the validity for specified Illumina basecalling data.  
#> O>     CollectIlluminaBasecallingMetrics (Picard)   Collects Illumina Basecalling metrics for a sequencing run.  
#> O>     CollectIlluminaLaneMetrics (Picard)          Collects Illumina lane metrics for the given BaseCalling analysis directory.
#> O>     ExtractIlluminaBarcodes (Picard)             Tool determines the barcode for each read in an Illumina lane.  
#> O>     IlluminaBasecallsToFastq (Picard)            Generate FASTQ file(s) from Illumina basecall read data.  
#> O>     IlluminaBasecallsToSam (Picard)              Transforms raw Illumina sequencing data into an unmapped SAM, BAM or CRAM file.
#> O>     MarkIlluminaAdapters (Picard)                Reads a SAM/BAM/CRAM file and rewrites it with new adapter-trimming tags.  
#> [long output truncated]

```

Then, we first prepare the reference sequence dictionary file by calling 
`Piccard`'s `CreateSequenceDictionary`:

1. Mount our input folder containing the reference fasta file as `/input` directory inside the container
2. Call `CreateSequenceDictionary` with both the input and output paths set to the mount point `/input/`.
This ensures that the output dictionary will be placed in the same folder as the reference fasta,
and will be available in our filesystem _outside_ the container

``` r

vol <- c("/home/input:/input")
cmd <- c("gatk", "CreateSequenceDictionary", "R=/input/hs1.fa.gz", "O=/input/hs1.dict")
docker_cli$container$run("broadinstitute/gatk", volumes = vol, cmd = cmd)

#> E> INFO	2023-02-13 03:16:12	CreateSequenceDictionary	
#> E> 
#> E> ********** NOTE: Picard's command line syntax is changing.
#> E> **********
#> E> ********** For more information, please see:
#> E> ********** 
#> [long output truncated]

```

Finally, we run the `HaplotypeCaller` in a similar way, mounting the same `/input` volume,
and making sure any output files are placed in the same folder:

``` r

vol <- c("/home/input:/input")
cmd <- c("gatk", "HaplotypeCaller", 
         "-R", "/input/hs1.fa.gz", 
         "-I", "/input/reads.sort.bam", 
         "-O", "/input/output.vcf")
docker_cli$container$run("broadinstitute/gatk", volumes = vol, cmd = cmd)

#> E> 03:17:34.629 INFO  NativeLibraryLoader - Loading libgkl_compression.so from jar:file:/gatk/gatk-package-4.3.0.0-local.jar!/com/intel/gkl/native/libgkl_compression.so
#> E> 03:17:34.731 INFO  HaplotypeCaller - ------------------------------------------------------------
#> E> 03:17:34.732 INFO  HaplotypeCaller - The Genome Analysis Toolkit (GATK) v4.3.0.0
#> E> 03:17:34.732 INFO  HaplotypeCaller - For support and documentation go to https://software.broadinstitute.org/gatk/
#> E> 03:17:34.732 INFO  HaplotypeCaller - Executing as root@b5bdd4aa690f on Linux v5.15.0-60-generic amd64
#> E> 03:17:34.732 INFO  HaplotypeCaller - Java runtime: OpenJDK 64-Bit Server VM v1.8.0_242-8u242-b08-0ubuntu3~18.04-b08
#> E> 03:17:34.732 INFO  HaplotypeCaller - Start Date/Time: February 13, 2023 3:17:34 AM GMT
#> [long output truncated]
```

Better make sure your read groups are sorted ;)

# Summary

`{stevedore}` makes it incredibly easy to call powerful containerized tools like 
`deepVariant` and `gatk` straight from `R` and making bioinformatic pipelines 
written in `R` more compact and easier to write, document, test and debug.

