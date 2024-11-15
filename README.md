## GlycoSiteMiner image and container

GlycoSiteMiner is a literature mining-based pipeline for extracting glycosylation sites from PubMed abstracts. The code for the pipeline is made available as a docker image that can be pulled using the following command.

```
docker pull rykahsay/glycositeminer
```

To start a container from the image, run the following command (make sure you change "/path/to/data" to the data path you want to use in your system)
```
docker run -itd -v /path/to/data:/data --name running_glycositeminer glycositeminer
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
/path/to/data/downloads/medline/
/path/to/data/downloads/gene_info/
/path/to/data/downloads/pubtator/
/path/to/data/downloads/glygen/
```

Next, run the following command to parse the *.xml.gz downloaded files under /path/to/data/downloads/medline/
and create medline extract files under /path/to/data/medline_extracts/. This is a parallelization wrapper script 
for another script named "extract-medline-data.py" and will spawn 10 "extract-medline-data.py"
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py &
```

When the 10 "extract-medline-data.py" are done and many files have been created under /path/to/data/medline_extracts/,
run the following command to parse downloaded file under /path/to/data/downloads/pubtator/
and create pubtator extract files under /path/to/data/pubtator_extracts/
```
docker exec -t running_glycositeminer python download.py -s wrap-extract-medline-data.py extract-pubtator-data.py &
```     





gene_info  glygen  medline  pubtator  tarballs



