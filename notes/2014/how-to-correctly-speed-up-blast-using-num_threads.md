---
[meta]
title = "How to correctly speed up blast using num_threads"
date = 2014-02-10
tags = ["bioinformatics", "blast", "genomics", "microbiome", "computational biology", "biology", "tools", "unix", "linux", "ncbi"]
description = "Have you ever wondered how much time you could save by using a 24 cores server to do a blast that is taking forever hours on your personal computer? Well I had; and others probably had too."
---
    #Disclaimer
    The tests applied here are far away from being 100% accurate, it's only purpose
    is give you an idea of how your blast+ will behave; but different hardware setup,
    length of sequences or blast+ parameters could dramatically change the outcomes.


Have you ever wondered how much time you could save by using a 24 cores server to do a blast that is taking forever 
hours on your personal computer? Well I had; and others probably had too.

One thing that everyone who thought about this, probably have noticed, is that the current blast+ from 
ncbi [doesn't use threads during all the process](http://seqanswers.com/forums/showthread.php?t=5752), indeed, if you 
run a blast -num_threds {put here your max cores}, you will see that most of the time, the processing bar of only one 
core will be full filled and the others will stay idle, waiting for something to do.

So as a experiment, I tested the same blast using different values for __num_threads.__

### Some considerations ###

  * My machine setup was 4 VPS from [digitalocean.com](https://www.digitalocean.com/?refcode=14945834b51b) , the specs are 8 cores with 16GB and a SSD disk;
  * I blasted with 1,2,4 and 8 threads;
  * My query dataset was 6.7k sequences of [mouse protein from ncbi refseq](ftp://ftp.ncbi.nih.gov/refseq/M_musculus/mRNA_Prot/mouse.protein.faa.gz);
  * The target database was built from full [zebrafish ncbi refseq protein](ftp://ftp.ncbi.nih.gov/refseq/D_rerio/mRNA_Prot/zebrafish.protein.faa.gz) records. (around 43k sequences);
  * After the first test I repeated all of them and got similar results;
  * The length of most sequences in both mouse and zebrafish fasta are between 300 and 900 amino acids;

The test command used in all four tests was:

<pre>
time blastp -query 6.7k_mouse.protein.faa -db zebrafish.protein.faa -num_threads X > output.log
</pre>


### Result ###

The raw and weird(what's up with those 3's?) result:
<pre>
1 thread  -> 193m10.105s
2 threads -> 143m54.688s -26%
4 threads -> 113m19.242s -41%
8 threads -> 103m25.137s -47%
</pre>

<center>![Alt text](/files/2014/test.png)</center>

From these values we can see that up to 4 threads we still get a good speed boost on blast+, but beyond 4 threads we 
would benefit little to almost nothing in terms of time reduction.

### Conclusion ###

Sounds pretty reasonable to me that doing multiple blasts at time is a better approach than let all cores to just one 
process. e.g. If you have a 8 core machine and 4 blasts to do, it should be a lot faster do all the four in 
parallel(assigning two threads to each) than do one at time using all 8 cores for each process.

The time in the parallel scenario would be around 143 minutes plus some extra time due the fact of four process write 
to disk at the same time, but unless you are reading this article 
from [the end of the 90's](http://www.tomshardware.com/reviews/15-years-of-hard-drive-history,1368-7.html), it's still 
way faster than 412 minutes of four blasts done in series. 
