---
[meta]
title = "Full text search with django and postgres"
date = 2020-02-08
tags = ["data engineering", "django", "python", "data engineer", "postgres", "full text search"]
description = "How to implement full text search in Django without using have bloated java software."
---
These days I was in need of a full text search. The cool <strike>kids</strike> teenagers in this 
block are Elastic Search and Sorl: they are fast, flexible, heavy on resource consumptions and
require java, almost all the things that I want for a pet project running in a $5 digital 
ocean droplet.

After discarding those options I was left with [Xapian](https://xapian.org/) and postgres full-text 
search, while xapian seems to be richer in features, I decided to start with postgres due it native 
integration with django and my modest requirement for this particular project.

###The project and it requirements

As you may have noticed, I'm running a [job board](https://www.voorjob.com). Voorjob is basically aggregating
jobs from lever.co and letting the user search through it. Currently I have around 25k jobs in the 
database, this number grows slow, for each 2 or 3 jobs added another job is closed. So yah, if I have 
taken the elastic search path on this one, It would have been a text-book over-engineering situation.

###The implementation

Since versions 9.4 postgres [added a few features](https://www.postgresql.org/docs/current/textsearch.html)
that allow full text search. Not much time later, Django mirrored those features in it
[postgres specific features](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/).   

To start using this new(ish) feature, I basically needed a `SearchVectorField` in my model
and some method to update this field with the _vectorized_ job description:

    from django.contrib.postgres.search import SearchVectorField, SearchVector

    class Job(models.Model):
        title = models.CharField(max_length=200, blank=True)
        location = models.CharField(max_length=50, blank=True)
        body = models.TextField(null=True)
        body_vector = SearchVectorField(null=True)

        def make_search_vector():
            self.body_vector=SearchVector('body')

        def save(self, *args, **kwargs):
            self.make_search_vector()
            super(Model, self).save(*args, **kwargs)
           

This approach works fine for something with few updates, like a job board, but if your application
have frequently updates, you should avoid this strategy and have some task that periodically populate
the vector:

    Job.objects.all().update(body_vector=SearchVector('body'))

or even better, you can do it directly with a postgres trigger by reading this 
[documentation](https://www.postgresql.org/docs/current/textsearch-features.html#TEXTSEARCH-UPDATE-TRIGGERS). 

### Querying for jobs

Now that you have your database ready it's time for querying it, let's see a didactic
version of voorjob search view:
    
    from django.contrib.postgres.search import SearchQuery

    class Index(ListView):
        model = Job
        paginate_by = 30
       
        def get_queryset(self):
            search = self.request.GET.get("search", None)
            queryset = Job.objects.all()

            if search:
                if '"' in search:
                    query = SearchQuery(search.replace('"', ''), search_type='phrase')
                else:
                    query = SearchQuery(search)
                queryset = queryset.filter(body_vector=query)
            else:
                queryset = queryset
    
            return queryset

I'm basically considering two kinds of queries here: **words presence** and **"exact expressions"**. Yes,
there are a few flaws in that logic, go ahead and sue me :D

There are much more things that can be improved, [django have support for weighted](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/search/#weighting-queries)
queries:

    vector =  SearchVector('title', weight='A') + SearchVector('body', weight='B')
    Job.objects.all().update(body_vector=vector)

This would ultimately return results in a better order, where a match in the title would give more
weight than a match in the body.

The query system is also more flexible allowing logical operations OR/AND and NOT. In a near future, 
I will improve the search of my job board and update this post to describe the changes.

### Performance

During development I used an I5 with 16GB of ram and a nice NVMe. Running queries
against the 25k jobs in my local machine is basically instantaneous.

When I moved the project to production (in the $5 droplet) things get waaaayyy more slow.

Running the mississippi benchmark I got the following result: 

 - ["django rest framework" search on /](https://www.voorjob.com/?search=%22django+rest+framework%22) ( 1 mississippi to scan 5K entries )
 - ["django rest framework" search on /full/](https://www.voorjob.com/full/?search=%22django+rest+framework%22) - ( 3 mississippi to scan 25K entries )
 
Not the best performance but works for now. This article will be updated to reflect any performance 
improvements.

Considering that my search needs are modest - _25k entries with an overage word count not much bigger
than this article_ - using postgres as backend for my full text search is working just fine for this
early stage MVP. Right now, I'm more interested in trying things and grow the board audience than 
giving my 20 daily user the fastest experience in the world.


### Update ( Feb 09, 2020)

Good news! I learned that I can add an index to my `SearchVectorField`:

    from django.contrib.postgres.indexes import GinIndex

    class Job(models.Model):
        class Meta:
            indexes = (GinIndex(fields=["body_vector"]),)

        title = models.CharField(max_length=200, blank=True)
        location = models.CharField(max_length=50, blank=True)
        body = models.TextField(null=True)
        body_vector = SearchVectorField(null=True)

        def make_search_vector():
            self.body_vector=SearchVector('body')

        def save(self, *args, **kwargs):
            self.make_search_vector()
            super(Model, self).save(*args, **kwargs)

Now the search time is down to 1 mississippi for all cases. Since my data is small,
the amount of memory used for this index is negligible.
