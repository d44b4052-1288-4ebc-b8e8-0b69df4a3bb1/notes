---
[meta]
title = "Downloading subsets of NCBI databases"
date = 2021-11-13
tags = ["bioinformatics", "python", "ncbi"]
description = ""
---
Let's say you want to download all protein assigned to Clostridum from the 
non-redundant protein database at NCBI. We can achieve that by using common 
tools available in every Linux/Mac system out there.

This theoretical pipeline (we will be using pipes so let's call it a pipeline) 
looks like this:

    stream/download file > decompress > parse records of interest > write to file

Now lets replace the actions by actual tools:

**stream/download**

To stream the data we can use curl:

```commandline
curl $URL
```
**decompress**

To decompress the stream of data from the previous command we can use gzip:
```commandline
gzip -d -
```

**parse records of interest and write to file**

To parse and write the output we can write a quick python script that reads 
from stdin and write only the records that contains the string of interest in 
the record header:

```python
#!python3
import sys
copy_is_on = False

filter_string = sys.argv[1]

with open("output.fa", "w", buffering=100000) as fh:
    while True:
        line = sys.stdin.readline()
        if line.startswith(">"):
            if filter_string in line.lower():
                copy_is_on = True
            else:
                copy_is_on = False

        if copy_is_on:
            fh.write(line)

```

Next we wrap everything into a single command:

```commandline
curl https://ftp.ncbi.nlm.nih.gov/blast/db/FASTA/nr.gz | gzip -d - | python3 filter.py clostridium
```