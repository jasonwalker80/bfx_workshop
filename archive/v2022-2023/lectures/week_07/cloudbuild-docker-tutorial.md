# Cloud Build Docker Tutorial

Let's build a docker container, but this time, we'll build it and host it in our cloud space, so that it's accessible when we try to run things there. 

1. Cloud Shell Workspace - First, let's set up our workspace:

Log in to a Cloud Shell Terminal: https://shell.cloud.google.com/?show=ide%2Cterminal

With Cloud Shell, you are automatically asked to authenticate with a WUSTL Key Google account.

If you are attempting this tutorial OUTSIDE of a Cloud Shell Terminal session, ex. local bash terminal, explicitly authenticate with your WUSTL Key account. 
```
gcloud auth login
```

Set the Google Project we are using for the remainder of this tutorial.
```
gcloud config set project icts-precision-health
```

Show the value for the environment variable `$USER`.
```
echo $USER
```

Some Google Cloud services do not allow characters like underscores that are common in WUSTL Key account names. 

Set a new "clean" shell variable to remove invalid characters:
```
USER_CLEAN=`echo $USER | sed 's/_/-/g'` && export USER_CLEAN
```

Create a directory and setup a cloudshell workspace using the new directory:
```
mkdir gatk-depth-filter-docker
```

```
cloudshell workspace gatk-depth-filter-docker
```

With the buttons at the top right, you can toggle between the Terminal (using `Open/Close Terminal`) and the Editor (using `Open/Close Editor`). You can also leave them both open at the same time.

In the Terminal, navigate into the newly created directory:
```
cd gatk-depth-filter-docker
``` 

2. Python Script - we need to pull in the python script that we want to include next to GATK in our container.

In the Terminal, use `wget` to copy the Python script to our workspace directory directly from GitHub.  
```
wget https://raw.githubusercontent.com/genome/docker-depth-filter/master/depth_filter.py
```

3. Dockerfile - the script that docker will use to create the image

Using the Editor, Save a file named `Dockerfile` in the `gatk-depth-filter-docker` workspace containing:
```
FROM broadinstitute/gatk:4.3.0.0
# make sure we're set up to securely pull packages from the repo
RUN curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# install a package for identity management that's useful on clusters and the cloud
RUN apt-get update && apt-get install -y libnss-sss && apt-get clean all
#now, go on with adding the tools/scripts we need
RUN pip install vcfpy pysam
COPY depth_filter.py /usr/bin/depth_filter.py
```

Note that this essentially the same thing we used in our local install, except that we added a step needed for securely installing software in the cloud.  Generally speaking, a Dockerfile should be platform independent, even though the steps around building them might differ slightly.

4. Cloud Build - create the docker image

On our local machines, we could just say "docker build" to build an image, then "docker push" to shoot it up to dockerhub (assuming you've created an account there and are logged in).  Here on GCP, we need to use our own repository/registry, and set up a config file to tell the build process how to push it there for later use.

Using the Editor, Save a file named `cloudbuild.yaml` in the `gatk-depth-filter-docker` workspace containing:
NOTE: REPLACE `$USER_CLEAN` WITH THE VALUE FROM THE ENVIRONMENT VARIABLE. This should be your WUSTL Key with `-` instead of `_` characters.
```
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-t', 'us-central1-docker.pkg.dev/icts-precision-health/bfx-workshop-repo/$USER_CLEAN-gatk-depth-filter-image:0.1', '.' ]
images:
- 'us-central1-docker.pkg.dev/icts-precision-health/bfx-workshop-repo/$USER_CLEAN-gatk-depth-filter-image:0.1'
```
Note that we've given it a unique name ($USER_CLEAN-gatk-depth-filter-image) and a version tag (0.1)

In the Terminal, submit the build:
```
gcloud builds submit --region=us-central1 --config cloudbuild.yaml
```

If you hit errors like `ERROR: (gcloud.builds.submit) parsing cloudbuild.yaml: while parsing a block collection`, check that your yaml file's spacing and indentation are exactly the same as the example above. 

You can follow your build's progress on the command line, or by looking at the "History" tab in Google Cloud Build: https://console.cloud.google.com/cloud-build/dashboard;region=us-central1?project=icts-precision-health

5. Docker Run - actually use your container

First, from a Terminal, we need to configure the Artifact Registry credentials for the region we intend to pull the Docker image from.
```
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Now, we can use the Docker `run` command to pull the image from the Artifact Registry and jump into the container once it's running with Docker.
```
docker run -it us-central1-docker.pkg.dev/icts-precision-health/bfx-workshop-repo/$USER_CLEAN-gatk-depth-filter-image:0.1 /bin/bash
```

# Germline Variant Detection
We will repeat the Germline Variant Detection steps from the local tutorial. 

However, this time we will perform all of the steps in the cloud including the execution of the depth_filter.py script and upload of the VCF back to a Cloud Bucket for use with IGV.

Please be sure you are using the Docker container terminal (in Cloud Shell) from Step 5 of the Cloud Build tutorial. If not, load the container using Step 5 in Cloud Shell now.

## GATK

```
gatk --list
```

```
gatk HaplotypeCaller --help
```
## Input
Use the cromwell-server API to retrieve the path to the HCC1395 Normal BAM file (ex. `alignment_exome.cwl.bam`).

Replace $BAM with the path returned from Cromwell. 

Example: `gs://icts-precision-health-cromwell-wf-exec/alignment_exome.cwl/d153b0da-ef5e-43ba-94dc-69e24311c83f/call-alignment/sequence_to_bqsr.cwl/a1da9f01-7014-411b-baef-e592bcf34cb6/call-index_bam/H_NJ-HCC1395-HCC1395_BL.bam`

## Output

We are now in a Docker container which does not inherit the environment from our Cloud Shell Terminal.

NOTE: Replace `$USER_CLEAN` in the output path below.

NOTE: Replace `$BAM` with the path returned by cromwell-server in the [DNA Alignment Workflow Tutorial](../week_06/bfx_workshop_06_alignment.md). 
```
gatk HaplotypeCaller --input $BAM --output gs://icts-precision-health-bfx-workshop-scratch/$USER_CLEAN/H_NJ-HCC1395-HCC1395_BL.vcf --reference gs://analysis-workflows-example-data/somatic_inputs/hla_and_brca_genes.fa
```
# Depth Filter

## Input

Our Python script does not accept `gs://` paths. We must stage the file to run locally in our Cloud Shell Terminal.

NOTE: Again, replace `$USER_CLEAN` with the value.
```
cd ~
mkdir week_07_gatk
cd week_07_gatk
gsutil cp gs://icts-precision-health-bfx-workshop-scratch/$USER_CLEAN/H_NJ-HCC1395-HCC1395_BL.vcf .
```

```
python /usr/bin/depth_filter.py --minimum_depth=30 H_NJ-HCC1395-HCC1395_BL.vcf H_NJ-HCC1395-HCC1395_BL H_NJ-HCC1395-HCC1395_BL.depth_filter.vcf
```

## Output

The file exists within our Docker container in Cloud Shell. Save it to a folder with your username inside the BFX Workshop scratch bucket.

NOTE: One more time, replace `$USER_CLEAN` with the actual value.
```
gsutil cp H_NJ-HCC1395-HCC1395_BL.depth_filter.vcf gs://icts-precision-health-bfx-workshop-scratch/$USER_CLEAN/
```

When finished with GATK and depth_filter.py, exit the Docker container in Cloud Shell Terminal returning to the original prompt.
```
exit
```

# IGV

Load the BAM and depth filtered VCF in IGV. Use the [Cloud Bucket browser](https://console.cloud.google.com/storage/browser?project=icts-precision-health) to find your files (or use `gsutil ls`) 

# Cleanup

When you're all done, do some cleanup (cloud storage isn't free!) From a Cloud Shell Terminal, remove your filder within the BFX Workshop scratch space:

NOTE: `$USER_CLEAN` should still be set in our Terminal environment from earlier.
```
gsutil rm -r gs://icts-precision-health-bfx-workshop-scratch/$USER_CLEAN/
```
