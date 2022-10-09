---
[meta]
title = "Writing a self-contained ETL pipeline with python"
date = 2020-01-25
tags = ["data engineering", "python", "data", "data engineer", "stream", "generator", "pulling", "pipeline",
"self-contained", "standalone", "executable"]
description = "For this tutorial we are going to create a self-contained small ETL pipeline for processing reddit posts in real time."
---

Python is an awesome language, one of the few things that bother me is not be able to bundle my code into a executable.
For as long as I can remember there were attempts to emulate this idea, mostly of them
didn't catch.

Since python 3.5 there is a new module in the standard library called zipapp that allow us to achieve
this behavior (with some limitations). This module generates a zip file that can bundle your
entire python project (including a virtual environment) and run it as an executable, like this:

    > ./your_project.pyz

It's practical. I have a few utilities built with this library and whenever I need to use them
in a new machine I just copy the "executable". No need to download the source from github and fiddle
with virtual environments.

This zipapp module is so cool that are companies using it to [build artifacts for deployment 
purpose](https://lincolnloop.com/blog/single-file-python-django-deployments/), and I'm also planning
to do it on [one of my projects](https://www.github.com/voorloopnul/pipetaxon/).

In this tutorial we are going to create a small ETL pipeline for processing reddit posts in real 
time, I will not write too much about the pipeline itself, to avoid ending with an extensive article.
But if you have any doubts feel free to ask in the comments.

### Step 0 - Project structure
Before you start, create a new project with the following structure:

    project/
        - pipeline.py
        - requirements.txt
        - __main__.py
        - build.sh

### Step 1 - Consuming a data stream
The first thing we need to do is consume a reddit endpoint as a stream of records, an explanation
about this block of code can be seen 
[in this article](http://voorloopnul.com/blog/abstracting-a-json-http-endpoint-into-a-continuous-data-stream/). 


    data_source = "https://www.reddit.com/r/all/new/.json"
    headers = {'User-agent': 'stream pipeline v.1'}
    
    def stream():
        buffer = []
        while True:
            time.sleep(.5)
            buffer = buffer[-1000:]
            r = requests.get(data_source, headers=headers)
            for record in r.json()["data"]["children"]:
                record = record["data"]
                if record["permalink"] not in buffer:
                    buffer.append(record["permalink"])
                    yield record

### Step 2 - Cleaning/Transforming the data

We are going to store only the `author_fullname`, `subreddit` and `title`, we need to remove
the other attributes, and rename author\_fullname to something less verbose like `author`. Since this
is an educational pipeline the fields and transformations were chosen in an arbitrary fashion:

    FIELDS = ["title", "subreddit", "author_fullname"]
    RENAME = {"author_fullname": "author"}

    def transform(record):
        # Clean entry
        key_list = list(record.keys())
        for key in key_list:
            if key not in FIELDS:
                del record[key]
    
        # Rename unwanted dictionary keys/value
        for old_key, new_key in RENAME.items():
            record[new_key] = record.pop(old_key)
    
        return record

### Step 3 - Load/Persisting

Here we are going to persist the data in SQLite database, it's a fine choice for a tutorial, and also
a good companion for small utilities built with zipapp module: 

    def load(record, conn):
        conn.execute("INSERT INTO COMPANY (TITLE, AUTHOR, SUBREDDIT) VALUES (:title, :author, :subreddit)", record)
        conn.commit()


### Step 4 - Gluing everything together

As usual, we need to glue our pipeline features together, we also use this space to create our
database, and to display some logging to track the data insertion:

    def create_database(conn):
        conn.execute("""
                CREATE TABLE COMPANY (ID INTEGER PRIMARY KEY,
                                      SUBREDDIT      CHAR(500)     NOT NULL,
                                      TITLE          CHAR(1000)    NOT NULL,
                                      AUTHOR         CHAR(500)     NOT NULL );
                 """)
        
    def run(db_name):
        conn = sqlite3.connect(db_name)
        create_database(conn)
        count = 0
        for record in stream():
            record = transform(record)
            load(record, conn)
            count += 1
            print(f"Records inserted in database: {count}",  end='\r')

### Step 5 - Zipapp entrypoint
            
For running our pipeline we need to create a \_\_main\_\_.py file, this will be the entrypoint
for our code. Every time you run the "executable" the \_\_main\_\_.py is called:

    """redpipe.
    
    Usage:
      redpipe.pyz <database>
    """
    
    from docopt import docopt
    import pipeline
    
    if __name__ == '__main__':
        arguments = docopt(__doc__, version='r.p.1')
        database_name = arguments["<database>"]
        pipeline.run(database_name)

### Step 6 - Making the pipeline self-contained

Now that we have our app ready we need a script to build it as a self-contained pipeline, it's going to bundle
the entire content of your requirements.txt and the pipeline code in a compressed zip file, with a custom
structure that allow it be executed as a single python file:
    
    #!/bin/bash
    rm -rf /tmp/build
    mkdir /tmp/build
    pip install -r requirements.txt --target /tmp/build
    cp -R . /tmp/build
    python -m zipapp -o build.pyz /tmp/build -p '/usr/bin/env python3'

If you glued everything correctly, you can now type:

    > ./build.sh

This will generate a new file "build.pyz". You can rename it to whatever you want, and execute it by typing:

    > ./build.pyz database_name.db
    
That's it! It is self-contained. You can distribute this pipeline to your family and friends or deploy somewhere else.

**Now the bad news.** You can't do that with a bunch of useful modules, pandas for instance will fail due numpy incompatibility,
django will fail for not being able to access some translation files (but flask + gunicorn will work fine). **If you stay with 
simpler packages built on top of python standard library you are probably safe.**

For things more complex you can try something like [PEX](https://github.com/pantsbuild/pex) or 
[shiv](https://github.com/linkedin/shiv), they will have a different approach but will create the 
illusion of a self-contained python app.

_PS. If you are having trouble gluing everything together you can take a look on the individual files:_

  - [requirements.txt](https://paste.ubuntu.com/p/gSbxKQgc7n/)
  - [pipeline.py](https://paste.ubuntu.com/p/GH4nqBHXcf/)
  - [\_\_main\_\_.py](http://paste.ubuntu.com/p/dv3g7R5yKz/)
  - [build.sh](https://paste.ubuntu.com/p/RGnt8cCz8X/)  
