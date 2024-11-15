## GlycoSiteMiner image and container

GlycoSiteMiner is a literature mining-based pipeline for extracting glycosylation sites from PubMed abstracts. The code for the pipeline is made available as a docker image that can be pulled using the following command.

```
docker pull rykahsay/glycositeminer
```

To start a container from the image, run the following command (make sure you change "/path/to/data" to the data path you want to use in your system)
```
docker run -itd -v /path/to/data:/data --name running_glycositeminer glycositeminer
```

## Download data generated in this study
Use the following commands to download and unpack data used by the pipeline
```
wget https://data.glygen.org/ftp/glycositeminer/tarballs/medline_extracts.tar.gz -O /path/to/data/downloads/medline_extracts.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/pubtator_extracts.tar.gz -O /path/to/data/downloads/pubtator_extracts.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/glygen.tar.gz -O /path/to/data/downloads/glygen.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/gene_info.tar.gz -O /path/to/data/downloads/gene_info.tar.gz --no-check-certificate
wget https://data.glygen.org/ftp/glycositeminer/tarballs/misc.tar.gz -O /path/to/data/downloads/misc.tar.gz --no-check-certificate

nohup tar xfz /path/to/data/downloads/medline_extracts.tar.gz -C /path/to/data/ &
nohup tar xfz /path/to/data/downloads/pubtator_extracts.tar.gz -C /path/to/data/ &
nohup tar xfz /path/to/data/downloads/glygen.tar.gz -C /path/to/data/ &
nohup tar xfz /path/to/data/downloads/gene_info.tar.gz -C /path/to/data/ &
nohup tar xfz /path/to/data/downloads/misc.tar.gz -C /path/to/data/ &
```

When this download/unpack is done, you should see the following file counts
```
18013 files under /path/to/data/medline_extracts/
16954 files under /path/to/data/pubtator_extracts/ 
   48 files under /path/to/data/glygen/ 
   10 files under /path/to/data/misc/
   7 files under /path/to/data/gene_info/ 
```




## Making entities 
The following command will make various entity type files under /path/to/data/entities/
```
docker exec -t running_glycositeminer python make-entities.py &
```


## Integrating entities 
The following command makes integrated entities and places them under /path/to/data/integrated/
```
docker exec -t running_glycositeminer python integrate-entities.py &
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
/path/to/data/medline/
/path/to/data/gene_info/
/path/to/data/pubtator/
/path/to/data/glygen/
```

Next, run the following command to parse the *.xml.gz downloaded files under /path/to/data/medline/
and create medline extract files under /path/to/data/medline_extracts/. This is a parallelization wrapper script 
for another script named "extract-medline-data.py" and will spawn 10 "extract-medline-data.py"
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py &
```

When the 10 "extract-medline-data.py" are done and many files have been created under /path/to/data/medline_extracts/,
run the following command to parse downloaded file under /path/to/data/pubtator/
and create pubtator extract files under /path/to/data/pubtator_extracts/
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py extract-pubtator-data.py &
```     






