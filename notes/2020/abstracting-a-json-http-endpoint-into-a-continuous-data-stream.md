---
[meta]
title = "Abstracting a json HTTP endpoint into a continuous data stream"
date = 2020-01-18
tags = ["data engineering", "python", "data engineer", "stream", "generator", "polling", "pipeline"]
description = "There are a few ways that can help you convert streams into batch, batch into streams and other things in between. The simplest way to convert a json endpoint into a stream is using a python generator and polling the data perpetually."
---

Data streams are cool, they poses a series of challenges that a batch usually don't (of course the
opposite also holds true). What about streams of batch? Different beast, different challenges but usually my last
preferred way of consuming data.

Fortunately there are a few ways that can help us convert streams into batch, batch into streams and 
other things in between.

Let's take for example the following endpoint: [http://reddit.com/r/all/new](). It works like an aggregator for reddit, posts
from every other subreddit show ups here at a high frequency (for subreddit standards) every ~5 seconds
the first 25 entries usually get replaced by 25 news entries, it gives us a throughput of 5 entries per second, it's
not much, but it's enough to start playing with streams.

The simplest way to convert a json endpoint into a stream is using a python generator and polling 
the data perpetually:
    
    def stream():
        while True:
            time.sleep(1)  # let's not be greedy
            r = requests.get(data_source, headers=headers)
            yield from (record["data"] for record in r.json()["data"]["children"])
    
Since the data is actually a batch snapshot, we will get a lot of duplicated records. If we 
increase the time.sleep() from `1` to `3` we are going to have less duplicates, if we increase 
to `6` we probably will not have duplicates at all, but we will also miss some records. Obviously 
there is no way of timing it properly to only get new records without missing a few others in the process...

To make sure that the records going into the stream are not duplicated we are going to use a buffer,
we could also leave this task to our pipeline/consumer but it would feel like __there is an abstraction missing somewhere__, 
so ideally we fix that right away in our stream maker:
      
    def stream():
        buffer = []
        while True:
            time.sleep(1)
            r = requests.get(data_source, headers=headers)
            for record in r.json()["data"]["children"]
                record = record["data"]
                if record["permalink"] not in buffer:
                    buffer.append(record["permalink"])
                    yield record
    
Now we only have unique records flowing through our stream, but this also creates a new problem:
the `buffer` will grow indefinitely and after a reasonable large amount of time our server will 
run out of memory.

But if you think about it, we actually only need to keep track of 25 permalinks, since they are supposed
to be unique and will never show up in a previous higher position we can just add one more line to make
sure that everything is streamed only once:

    buffer = buffer[-25:]
    
__UPDATE!__ Actually due the voting nature of reddit, articles in /new can sometimes regain a 
few positions, but they never stay long in the first page. So to make sure no duplicates occur we 
will rise the buffer to an arbitrary safe number:

    buffer = buffer[-1000:]
    
That's it, we just have tamed our buffer! The final code with all imports and a consumer function should
look more or less like this:

    import time
    import requests

    data_source = "https://www.reddit.com/r/all/new/.json"
    headers = {'User-agent': 'stream pipeline v.1'}
   
    def stream():
        buffer = []
        while True:
            time.sleep(1)
            buffer = buffer[-25:]
            r = requests.get(data_source, headers=headers)
            for record in r.json()["data"]["children"]:
                record = record["data"]
                if record["permalink"] not in buffer:
                    buffer.append(record["permalink"])
                    yield record
    
    for entry in stream():
        print(entry["title"])

If you use this code to create a pipeline there is still a problem that will arise when your processing time
takes longer than 5 seconds (for each "batch" of 25 records). Since the yield will only unblock after 
it get depleted by the consumer, you need to make sure that your processing is actually faster than 
the ability of reddit to produce new entries, otherwise you risk to lose some entries until your next http
request fires up.

