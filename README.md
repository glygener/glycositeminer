# GlycoSiteMiner image and container

GlycoSiteMiner is a literature mining-based pipeline for extracting glycosylation sites from PubMed abstracts. The code for the pipeline is made available as a docker image that can be pulled using the following command.


```
docker pull glygen/glycositeminer:1.0.0
```

After pulling the image, create a data directory and set an env variable that will contain the path to your data directory. In the example shown below, 
the directory "/data/glycositeminer/" is used a a data directory.
```
mkdir -p /data/glycositeminer/
export DATA_PATH=/data/glycositeminer/
```

To start a container from the image, run the following command
```
docker run -itd -v $DATA_PATH:/data --name running_glycositeminer glygen/glycositeminer:1.0.0
```


### Step-1: download data generated in this study
Use the following command to download and unpack data used by the pipeline. 
```
nohup docker exec -t running_glycositeminer python download-pipeline-data.py &
```
When the process started by the above command is done, you should see the following file counts
```
 9311 files under $DATA_PATH/medline_abstracts/pmid.*.txt
16954 files under $DATA_PATH/pubtator_extracts/pmid.*.txt
27933 files under $DATA_PATH/llm_entities/*/llm.*.json
  124 files under $DATA_PATH/glygen/* 
   20 files under $DATA_PATH/misc/*
    1 file under $DATA_PATH/gene_info/*
    1 file under $DATA_PATH/pdb/*
```
NOTE: if you wish to complile these dataset files from their original source, instructions are given at the bottom of this README.


### Step-2: making PubTator entities 
The following commands will use downloaded PubTator files to make PubTator entity type files under "$DATA_PATH/pubtator_entities/".  
```
nohup docker exec -t running_glycositeminer python make-pubtator-entities.py &
```

After number of entities should be as follows
```
3329 gene entities ($DATA_PATH/pubtator_entities/gene.*.json)
3930 species entities ($DATA_PATH/pubtator_entities/species.*.json)
```


### Step-3: mapping LLM entities step 1
This step should create $\color{red}{3947}$ files under "$DATA_PATH/sites/".
```
nohup docker exec -t running_glycositeminer python map-llm-entities-step-1.py &
```

### Step-4: mapping LLM entities step 2
This step should create $\color{red}{3350}$ files under "$DATA_PATH/canons/".
```
nohup docker exec -t running_glycositeminer python map-llm-entities-step-2.py &
```


### Step-5: creating match sites
The command given below will create $\color{red}{4261}$ sequence-specific match sites in "$DATA_PATH/match_sites/sites.csv".
```
nohup docker exec -t running_glycositeminer python make-match-sites.py &
```


### Step-6: creating samples
The command given below will generate two files --  "$DATA_PATH/samples/samples_all.csv" containing $\color{red}{4261}$ both labeled and unlabelled samples, and "$DATA_PATH/samples/samples_labeled.csv" containing $\color{red}{886}$ positive and $\color{red}{238}$ negative labeled samples.
```
nohup docker exec -t running_glycositeminer python make-samples.py &
```


### Step-7: model validation
This step will run 10-fold cross validation using the samples in "$DATA_PATH/samples/samples_labeled.csv", and the
output files will be under "$DATA_PATH/validation/". The "performance.csv" file contains performance 
output values for each run for both SVM and MLP classifiers, and the confusion matrix values are in the file
"confusion_matrix.json". The are also two PNG files, "roc.png" and "cm.png", showing the ROC curves and confusion 
matrix respectively.
```
nohup docker exec -t running_glycositeminer python run-cross-validation.py &
```


### Step-8: tuning the decision threshold for class prediction
As described in the paper, these commands given below are used to find optimal threshold on the class probabilities that is 
suitable for our application. The output of the first command is saved in "$DATA_PATH/tuning/tuning.json", 
and the second command generates a PNG file "$DATA_PATH/tuning/balanced_accuracy.png" and cutoffs file "$DATA_PATH/tuning/cutoff.json" which contains cutoff values to be used when making predictions. 
```
nohup docker exec -t running_glycositeminer python tuning-step-1.py &
```

You need to wait until the process started by the above command is finished.
```
nohup docker exec -t running_glycositeminer python tuning-step-2.py &
```


### Step-9: building final models
Using all the samples in "$DATA_PATH/samples/samples_labeled.csv", this step creates final modesl for both
SVM and MLP classifiers and saves the models under "$DATA_PATH/models/".
```
docker exec -t running_glycositeminer python make-models.py 
```


### Step-10: making predictions
We can now apply the models to all samples "$DATA_PATH/samples/samples_all.csv" to make predictions. The output of the command below
is saved in "$DATA_PATH/predicted/predicted.csv". As reported in the paper, this file contains a total of $\color{red}{3118}$ predicted sites. This script also outputs stat files "$DATA_PATH/predicted/stats.txt" giving the number of sites predicted for each species.
```
docker exec -t running_glycositeminer python make-predictions.py 
```


# DOWNLOADING ORIGINAL DATA
To download and process original data, follow the instructions given below.

### GlyGen downloads
Run the following command and the downloaded files will be saved under "$DATA_PATH/glygen/".
```
nohup docker exec -t running_glycositeminer python download-glygen.py &
```

### Gene Info downloads
Run the following command and the downloaded files will be saved under "$DATA_PATH/gene_info/".
```
nohup docker exec -t running_glycositeminer python download-gene-info.py &
```

### PubMed downloads
Run the following command to download PubMed baseline xml files for 2024 which have file indexes starting from "1" to "1219". To find out the 
start and end values of the baseline files, visit "https://ftp.ncbi.nlm.nih.gov/pubmed/baseline/" and look at the first and last *.xml.gz files. 
For year 24, since the first and last files are "pubmed24n0001.xml.gz" and "pubmed24n1219.xml.gz", the start and end file indexes are "1" and "1219". 
The downloaded files will be saved under "$DATA_PATH/medline/".
```
nohup docker exec -t running_glycositeminer python download-medline.py -c baseline -y 24 -s 1 -e 1219 &
```
Similarly, for the PubMed updatefiles, visit "https://ftp.ncbi.nlm.nih.gov/pubmed/updatefiles/" and you will find out first and last files 
are "pubmed24n1220.xml.gz" and "pubmed24n1600.xml.gz".
```
nohup docker exec -t running_glycositeminer python download-medline.py -c updatefiles -y 24 -s 1220 -e 1600 &
```

### PubTator downloads
Run the following command and the downloaded files will be saved under "$DATA_PATH/pubtator/".
```
nohup docker exec -t running_glycositeminer python download-pubtator.py &
```

### Making PubMed and PubTator extracts
Next, run the following command to parse the *.xml.gz downloaded files under $DATA_PATH/medline/
and create medline extract files under $DATA_PATH/medline_extracts/. This is a parallelization wrapper script 
for another script named "extract-medline-data.py" and will spawn 10 "extract-medline-data.py"
```
nohup docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py &
```

When the 10 "extract-medline-data.py" are done and many files have been created under $DATA_PATH/medline_extracts/,
run the following command to parse downloaded file under $DATA_PATH/pubtator/
and create pubtator extract files under $DATA_PATH/pubtator_extracts/
```
nohup docker exec -t running_glycositeminer python download.py -s extract-pubtator-data.py &
```     






