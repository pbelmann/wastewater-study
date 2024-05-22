# Metagenomics-tk setup for Global Wastewater Study

The following instructions explain how we setup the metagenomics-tk pipeline for the re-analysis of a global wastewater study.

## Per Sample Analysis

### Prerequisites

This setup has been tested on a SLURM system where all VMs are based on Ubuntu.
You will need to install java on the master node.

### 1. Checkout the repo and download all necessary assets

1. Checkout `git clone  --depth 1 --branch  <tag name> git@github.com:pbelmann/metagenomics-tk.git`

### 2. Download cplex binary

This part only works if you have access to a private bucket hosted on the Bielefeld de.NBI Cloud site.
If this is not the case, you can download the cplex binary from the IBM website.
Note that you will need to register with IBM to obtain an academic license for CPLEX.


1. Change directory to metagenomics-tk: `cd metagenomics-tk`

2. Create a file named `credentials` for s5cmd with the following content:

```
[default]
aws_access_key_id=
aws_secret_access_key=
```

3. Place the cplex binary in cplex/docker: 
   `./bin/s5cmd --credentials-file credentials  --endpoint-url https://openstack.cebitec.uni-bielefeld.de:8080 cp s3://wastewater-assets/cos_installer-1.bin  cplex/docker/`

4. Update the buildScript.sh script in the cplex directory, by setting the `DOCKERILE_FOLDER` variable. It should point to the folder where the Dockerfile is placed.
   Example: `/vol/spool/final/metagenomics-tk/cplex/docker`


### 3. Configure metagenomics-tk

1. If you want to upload your results to Object storage, you will have to create AWS config file which looks like this:
   
```
aws {

  accessKey = ""
  secretKey = ""


    client {

      s_3_path_style_access = true
      maxParallelTransfers = 28 
      maxErrorRetry = 10
      protocol = 'HTTPS'
      endpoint = 'https://openstack.cebitec.uni-bielefeld.de:8080'
      signerOverride = 'AWSS3V4SignerType'
    }
}
```

2. If you have not created a credentials file in previous steps, create a file with the following format and save it in a folder
that is shared by all worker nodes.

```
[default]
aws_access_key_id=
aws_secret_access_key=
```

3. In case you want to to run more jobs in parallel, you can update the Nextflow queue size (`queueSize`) in the nextflow.config file. 

### 4. Download and prepare GTDB for fragment recruitment

1. Download GTDB

```
mkdir -p gtdb && wget https://openstack.cebitec.uni-bielefeld.de:8080/databases/gtdbtk_r214_data.tar.gz  -O - | tar -xzvf - -C gtdb
```  

2. Prepare paths file

```
echo "PATH" > paths.tsv && find gtdb -name "*.fna.gz" | xargs -I {} readlink -f {} >> paths.tsv
```

### 5. Execute the per sample mode

1. Install nextflow
`cd metagenomics-tk && make nextflow`

2. In addition to the main Nextflow run command you can optionaly add the following parameter:

Specify `-with-weblog http://localhost:8000/run/<token-id>/` if you want to use a logging system like nf-tower or TraceFlow(https://github.com/vktrrdk/nextflowAnalysis).

Final command:

```
./nextflow -c AWS run main.nf \
    -ansi-log false -profile slurm -resume -entry wFullPipeline -params-file CONFIG \
    --input.SRA.S3.path=SAMPLES \ 
    --steps.annotation.mmseqs2.kegg.database.download.s5cmd.keyfile=S5CMD_CREDENTIALS \
    --steps.fragmentRecruitment.mashScreen.genomes=FRAGMENT_RECRUITMENT \
    --smetana_image=pbelmann/metabolomics:0.1.0  --carveme_image=pbelmann/metabolomics:0.1.0 \
    --steps.metabolomics.beforeProcessScript=CPLEX  --steps.metabolomics.carveme.additionalParams='' 
```

where
  * AWS is file that should point to a file containing AWS credentials.
  * CONFIG is pointing to one of the config files in the "per_sample" folder. Example: https://raw.githubusercontent.com/pbelmann/wastewater-study/main/config/per_sample/bi/fullPipeline_illumina_nanpore_without_aggregate.yml   
  * SAMPLES is file containing SRA ids. It should be one of the files in the datasets folder https://raw.githubusercontent.com/pbelmann/wastewater-study/main/datasets/test-1.tsv?token=GHSAT0AAAAAABVMSFLS2YMOP6RIHMWK6E2WZH7Q5IQ
  * S5CMD_CREDENTIALS is a file containing credentials for S5CMD.
  * FRAGMENT_RECRUITMENT is a file containing a list of genomes. It 
  * CPLEX should point to the full path of the cplex build script (i.e. /path/to/cplex/buildScript.sh)

### 6. Postprocess

Once the run is finished,
 * collect the stats produced by the trace file and commit them with the correct name pattern.
 * fill in the form in the google spread sheet.

## Aggregation

In order to execute the aggregation mode you will have to follow the instructions of step 1,2 and 3 of the per sample run mode.

### 1. Execute the aggregation mode

1. Install nextflow
`cd metagenomics-tk && make nextflow`

2. In addition to the main Nextflow run command you can optionaly add the following parameter:

Specify `-with-weblog http://localhost:8000/run/<token-id>/` if you want to use a logging system like nf-tower or TraceFlow(https://github.com/vktrrdk/nextflowAnalysis).

Final command:

```
./nextflow -c AWS run main.nf \
    -profile slurm -resume -entry wAggregatePipeline \
    -params-file https://raw.githubusercontent.com/pbelmann/wastewater-study/main/config/aggregate/fullPipelineAggregate.yml \
    --smetana_image=pbelmann/metabolomics:0.1.0 \
    --steps.cooccurrence.beforeProcessScript=CPLEX \
    --output=OUTPUT
```

where
  * AWS is file that should point to a file containing AWS credentials. 
  * CPLEX should point to the full path of the cplex build script (i.e. /path/to/cplex/buildScript.sh)
  * OUTPUT points to an output directory or S3 bucket if available.
 
## Best Practices

* In case the pipeline stops, please first delete all sample keys in the output S3 bucket and then resume the pipeline.

