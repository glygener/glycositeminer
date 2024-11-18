## GlycoSiteMiner image and container

GlycoSiteMiner is a literature mining-based pipeline for extracting glycosylation sites from PubMed abstracts. The code for the pipeline is made available as a docker image that can be pulled using the following command.

```
docker pull glygen/glycositeminer
```

After pulling the image, create a data directory and set an env variable that will contain the path to your data directory. In the example shown below, 
the directory "/data/glycositeminer/" is used a a data directory.
```
mkdir -p /data/glycositeminer/
export DATA_PATH=/data/glycositeminer/
```

To start a container from the image, run the following command
```
docker run -itd -v $DATA_PATH:/data --name running_glycositeminer glygen/glycositeminer
```


## Step-1: download data generated in this study
Use the following commands to download and unpack data used by the pipeline
```
mkdir -p $DATA_PATH/downloads

wget https://data.glygen.org/ftp/glycositeminer/tarballs/medline_extracts.tar.gz -O $DATA_PATH/downloads/medline_extracts.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/pubtator_extracts.tar.gz -O $DATA_PATH/downloads/pubtator_extracts.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/glygen.tar.gz -O $DATA_PATH/downloads/glygen.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/gene_info.tar.gz -O $DATA_PATH/downloads/gene_info.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/misc.tar.gz -O $DATA_PATH/downloads/misc.tar.gz --no-check-certificate

nohup tar xfz $DATA_PATH/downloads/medline_extracts.tar.gz -C $DATA_PATH &
nohup tar xfz $DATA_PATH/downloads/pubtator_extracts.tar.gz -C $DATA_PATH &
nohup tar xfz $DATA_PATH/downloads/glygen.tar.gz -C $DATA_PATH &
nohup tar xfz $DATA_PATH/downloads/gene_info.tar.gz -C $DATA_PATH &
nohup tar xfz $DATA_PATH/downloads/misc.tar.gz -C $DATA_PATH &
```

When this download/unpack is done, you should see the following file counts
```
18013 files under $DATA_PATH/medline_extracts/
16954 files under $DATA_PATH/pubtator_extracts/ 
   43 files under $DATA_PATH/glygen/ 
   9 files under $DATA_PATH/misc/
    7 files under $DATA_PATH/gene_info/ 
```


## Step-2: making and integratig entities 
The following commands will make various entity type files under "$DATA_PATH/entities/" and integrate them under "$DATA_PATH/integrated/". The second command should be executed after
the first finishes. There are 9311 such interated entity files and one can parse the objects in these files and find 5424 "match sites" as described in the paper.  
```
docker exec -t running_glycositeminer python make-entities.py 
docker exec -t running_glycositeminer python integrate-entities.py 
```

## Step-3: creating labeled samples
Out of the 5424 "match sites" contained in the integrated entity files under "$DATA_PATH/integrated/", 
the command given below will generate labeled samples and save them in "$DATA_PATH/samples/samples_labeled.csv". 
As reportd in the paper, this file will contain 872 positive and 354 negative samples. The criteria for labeling 
these samples is described in the paper.
```
docker exec -t running_glycositeminer python make-labeled-samples.py 
```


## Step-4: model validation
This step will run 10-fold cross validation using the samples in "$DATA_PATH/samples/samples_labeled.csv", and the
output files will be under "$DATA_PATH/validation/". The "performance.csv" file contains performance 
output values for each run for both SVM and MLP classifiers, and the confusion matrix values are in the file
"confusion_matrix.json". The are also two PNG files, "roc.png" and "cm.png", showing the ROC curves and confusion 
matrix respectively.
```
docker exec -t running_glycositeminer python run-cross-validation.py 
```


## Step-5: tuning the decision threshold for class prediction
As described in the paper, these commands given below find optimal threshold on the class probabilities that is 
suitable for our application. The output of the first command is saved in "$DATA_PATH/tuning/tuning.json", 
and the second command generates a PNG file "$DATA_PATH/tuning/balanced_accuracy.png". You need to wait until 
the first command is finished.
```
docker exec -t running_glycositeminer python tuning-step-one.py &
docker exec -t running_glycositeminer python tuning-step-two.py
```

## Step-6: building final models
Using all the samples in "$DATA_PATH/samples/samples_labeled.csv", this step creates final modesl for both
SVM and MLP classifiers and saves the models under "$DATA_PATH/models/".
```
docker exec -t running_glycositeminer python build-models.py 
```


## Step-7: creating unlabeled samples
This step makes unlabled samples corresponding to the 5424 "match sites" and saves them under "$DATA_PATH/samples/samples_unlabeled.csv"
```
docker exec -t running_glycositeminer python make-unlabeled-samples.py 
```

## Step-8: making predictions
We can now apply the models to the unlabeled samples "$DATA_PATH/samples/samples_unlabeled.csv" to make predictions. The output of the command below
is saved in "$DATA_PATH/predicted/predicted.csv". As reported in the paper, this file contains a total of 3268 predicted sites.
```
docker exec -t running_glycositeminer python make-predictions.py 
```




## Using the container to download raw or original data 
You need to perform this step only if you want to perform data processing from the scratch. Otherwise, you can skip to the next step to download the data we have already processed. To download the original data, use the following commands.

```
docker exec -t running_glycositeminer python download.py -s medline &
docker exec -t running_glycositeminer python download.py -s gene_info &
docker exec -t running_glycositeminer python download.py -s pubtator &
docker exec -t running_glycositeminer python download.py -s glygen &
```

Once the above four processes finish, there will be downloaded files under the following folders
```
$DATA_PATH/medline/
$DATA_PATH/gene_info/
$DATA_PATH/pubtator/
$DATA_PATH/glygen/
```

Next, run the following command to parse the *.xml.gz downloaded files under $DATA_PATH/medline/
and create medline extract files under $DATA_PATH/medline_extracts/. This is a parallelization wrapper script 
for another script named "extract-medline-data.py" and will spawn 10 "extract-medline-data.py"
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py &
```

When the 10 "extract-medline-data.py" are done and many files have been created under $DATA_PATH/medline_extracts/,
run the following command to parse downloaded file under $DATA_PATH/pubtator/
and create pubtator extract files under $DATA_PATH/pubtator_extracts/
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py extract-pubtator-data.py &
```     






