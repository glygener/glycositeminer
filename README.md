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
You need to perform this step only if you want to perform data processing from the scratch. Otherwise, you can skip to the next step to download the data we have already processed. The original or raw data that need to be downloaded  you will needThe following datasets were downloa


