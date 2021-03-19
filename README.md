# Refgenie bowtie indexes

This repository contains necessary files to build and archive reference genome assets to serve with [`refgenieserver`](https://github.com/databio/refgenieserver). The whole process is scripted, starting from this repository. From here, we download the input data (FASTA files), use `refgenie build` to create all of these assets in a local refgenie instance, and then use `refgenieserver archive` to build the server archives, and finally serve them with a refgenieserver instance by calling `refgenieserver serve`.

# How to add an asset

1. Add a line for each genome in [genome_descriptions.csv](asset_pep/genome_descriptions.csv).
2. Add a line for each asset in [assets.csv](asset_pep/assets.csv).
3. Add any required build inputs (links to files, parameters, etc) to [recipe_inputs.csv](asset_pep/recipe_inputs.csv)

# Deploying assets onto the server

## Setup

In this guide we'll use environment variables to keep track of where stuff goes.

- `BASEDIR` points to our parent folder where we'll do all the building/archiving
- `GENOMES` points to pipeline output (referenced in the project config)
- `REFGENIE_RAW` points to a folder where the downloaded raw files are kept
- `REFGENIE` points to the refgenie config file
- `REFGENIE_ARCHIVE` points to the location where we'll store the actual archives

```
export SERVERNAME=btref.databio.org
export BASEDIR=$PROJECT/deploy/$SERVERNAME
export BASEDIR=$HOME/refgenie_test # For local testing

export GENOMES=$BASEDIR/genomes
export REFGENIE_RAW=$BASEDIR/refgenie_$SERVERNAME
export REFGENIE=$BASEDIR/$SERVERNAME/config/refgenie_config.yaml
export REFGENIE_ARCHIVE=$GENOMES/archive
mkdir -p $BASEDIR
cd $BASEDIR
```

To start, clone this repository:

```
cp -R $CODE/$SERVERNAME . # For local testing
<!-- git clone git@github.com:refgenie/$SERVERNAME.git -->
```

## Step 1: Download input files

Many of the assets require some input files, and we have to make sure we have those files locally. In the `recipe_inputs.csv` file, we have entered these files as remote URLs, so the first step is to download them. We have created a subproject called `getfiles` for this: To programmatically download all the files required by `refgenie build`, run from this directory using [looper](http://looper.databio.org):

```
cd $SERVERNAME
mkdir -p $REFGENIE_RAW
looper run asset_pep/refgenie_build_cfg.yaml -p local --amend getfiles --sel-attr asset --sel-incl fasta
```

Check the status with `looper check --use-pipesat`:

*`--use-pipesat` option is required as of early 2021. Might not be required if you're running later on.*

```
looper check asset_pep/refgenie_build_cfg.yaml --amend getfiles --sel-attr asset --sel-incl fasta --use-pipestat
```

## Step 2: Refgenie genome configuration file initialization

This repository comes with a genome configuration file already defined in [`\config`](config) directory, but if you have not initialized refgenie yet or want to start over, then first you can initialize the config like this:

```
refgenie init -c config/refgenie_config.yaml -f $GENOMES -u http://awspds.refgenie.databio.org/refgenomes.databio.org/ -a $GENOMES/archive -b refgenie_config_archive.yaml
```

## Step 3: Build assets

Once files are present locally, we can run `refgenie build` on each asset specified in the sample_table (`assets.csv`). We have to submit fasta assets first:

```
looper run asset_pep/refgenie_build_cfg.yaml -p bulker_local --sel-attr asset --sel-incl fasta
```

This will create one job for each *asset*. Monitor job progress with: 

```
grep CANCELLED ../genomes/submission/*.log
ll ../genomes/submission/*.log
grep error ../genomes/submission/*.log
grep maximum ../genomes/submission/*.log

ll ../genomes/data/*/*/*/_refgenie_build/*.flag
ll ../genomes/data/*/*/*/_refgenie_build/*failed.flag
ll ../genomes/data/*/*/*/_refgenie_build/*completed.flag
ll ../genomes/data/*/*/*/_refgenie_build/*running.flag
ll ../genomes/data/*/*/*/_refgenie_build/*completed.flag | wc -l
cat ../genomes/submission/*.log
```

To run all the asset types:

```
looper run asset_pep/refgenie_build_cfg.yaml -p bulker_local
```

## Step 4. Archive assets

Assets are built locally now, but to serve them, we must archive them using `refgenieserver`. The general command is `refgenieserver archive -c <path/to/genomes.yaml>`. Since the archive process is generally lengthy, it makes sense to submit this job to a cluster. We can use looper to do that. 

To start over completely, remove the archive config file with: 

``` 
rm config/refgenie_config_archive.yaml
```

Then submit the archiving jobs with `looper run`

```
looper run asset_pep/refgenieserver_archive_cfg.yaml -p bulker_local --sel-attr asset --sel-incl fasta
```

Check progress with:

```
ll ../genomes/archive_logs/submission/*.log
grep Wait ../genomes/archive_logs/submission/*.log
grep Error ../genomes/archive_logs/submission/*.log
cat ../genomes/archive_logs/submission/*.log
```

## Step 5. Upload archives to S3

Now the archives should be built, so we'll sync them to AWS. Use the refgenie credentials (here added with `--profile refgenie`, which should be preconfigured with `aws configure`)


```
aws s3 sync $REFGENIE_ARCHIVE s3://awspds.refgenie.databio.org/refgenomes.databio.org/ --profile refgenie
```

## Step 6. Deploy server 

Now everything is ready to deploy. If using refgenieserver directly, you'll run `refgenieserver serve config/refgenieserver_archive_cfg`. We're hosting this repository on AWS and use GitHub Actions to trigger  trigger deploy jobs to push the updates to AWS ECS whenever a change is detected in the config file. 

```
ga -A; gcm "Deploy to ECS"; gpoh
```
