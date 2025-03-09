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
18013 files under $DATA_PATH/medline_extracts/ (glycosylation terms containing sentences extracted from PubMed abstracts)
16954 files under $DATA_PATH/pubtator_extracts/ (extracts from PubTator archive)
   43 files under $DATA_PATH/glygen/ (reference dataset files downloaded from data.glygen.org)
    9 files under $DATA_PATH/misc/ (misc files used by the pipeline)
    1 files under $DATA_PATH/gene_info/ (file used to map Gene IDs to Ensembl locus) 
```
NOTE: if you wish to complile "medline_extracts.tar.gz" and "pubtator_extracts.tar.gz" starting from original PubMed and Putator downloads,
instructions are given at the bottom of this README.


### Step-2: making entities 
The following commands will use downloaded files to make various entity type files under "$DATA_PATH/entities/".  
```
nohup docker exec -t running_glycositeminer python make-entities.py &
```

After number of entities should be as follows
```
9311 site entities ($DATA_PATH/entities/site.*.json)
9311 glyco entities ($DATA_PATH/entities/glyco.*.json)
5917 gene entities ($DATA_PATH/entities/gene.*.json)
4593 extragene entities ($DATA_PATH/entities/extragene.*.json)
7260 species entities ($DATA_PATH/entities/species.*.json)
```


### Step-3: integratig entities 
The entities created should be integrated using the command given below. This step should create 9311 files under "$DATA_PATH/integrated/".
```
nohup docker exec -t running_glycositeminer python integrate-entities.py &
```


### Step-4: creating labeled samples
Out of the 5424 "match sites" contained in the integrated entity files under "$DATA_PATH/integrated/", 
the command given below will generate labeled samples and save them in "$DATA_PATH/samples/samples_labeled.csv". 
As reportd in the paper, this file will contain 872 positive and 354 negative samples. The criteria for labeling 
these samples is described in the paper.
```
nohup docker exec -t running_glycositeminer python make-labeled-samples.py &
```


### Step-5: model validation
This step will run 10-fold cross validation using the samples in "$DATA_PATH/samples/samples_labeled.csv", and the
output files will be under "$DATA_PATH/validation/". The "performance.csv" file contains performance 
output values for each run for both SVM and MLP classifiers, and the confusion matrix values are in the file
"confusion_matrix.json". The are also two PNG files, "roc.png" and "cm.png", showing the ROC curves and confusion 
matrix respectively.
```
nohup docker exec -t running_glycositeminer python run-cross-validation.py &
```


### Step-6: tuning the decision threshold for class prediction
As described in the paper, these commands given below are used to find optimal threshold on the class probabilities that is 
suitable for our application. The output of the first command is saved in "$DATA_PATH/tuning/tuning.json", 
and the second command generates a PNG file "$DATA_PATH/tuning/balanced_accuracy.png". 
```
nohup docker exec -t running_glycositeminer python tuning-step-one.py &
```

You need to wait until the process started by the above command is finished.
```
nohup docker exec -t running_glycositeminer python tuning-step-two.py &
```


### Step-7: building final models
Using all the samples in "$DATA_PATH/samples/samples_labeled.csv", this step creates final modesl for both
SVM and MLP classifiers and saves the models under "$DATA_PATH/models/".
```
docker exec -t running_glycositeminer python build-models.py 
```


### Step-8: creating unlabeled samples
This step makes unlabled samples corresponding to the 5424 "match sites" and saves them under "$DATA_PATH/samples/samples_unlabeled.csv"
```
docker exec -t running_glycositeminer python make-unlabeled-samples.py 
```

### Step-9: making predictions
We can now apply the models to the unlabeled samples "$DATA_PATH/samples/samples_unlabeled.csv" to make predictions. The output of the command below
is saved in "$DATA_PATH/predicted/predicted.csv". As reported in the paper, this file contains a total of 3268 predicted sites.
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






