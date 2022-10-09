---
[meta]
title = "Django tips for real life applications"
date = 2019-12-22
tags = ["django", "uuid", "cached_property", "cbv", "primary key", "object permission", "object owner", "production", "cache", "slug"]
description = "Real life django applications often have requirements that are not well advertised through mostly django tutorials around the internet. Here I share a few tips that may help you build a better django app."
---

### Summary

Real life django applications often have requirements that are not well advertised through mostly 
django tutorials around the internet. Here I share a few tips that may help you build a better django
app.

## Using UUIDs instead of IDs to reference objects.

Let's say for example that you are building an app that are going to disrupt the finance market, maybe
it is bitcoins, or you are going to fix the banking system (please do it).

Your user logIn, do a transaction and see something like this:

   > http://financeapp.com/transaction/37/
    
The user will realize that so far only 37 transactions where made on your application, and to be honest, he 
will probably not care about that, but depending of your audience, hiding this information could be useful.
It's also one information that you may want hide from current competitors or even hide from the press if
they have an eye on you.
  
The standard way to hide those sequential ID's is replacing them by an UUID, something like this:
    
   > http://financeapp.com/transaction/e12616c3-d81d-47a4-b01a-30d785be5251/
 
The code necessary to achieve that in django is really simple and almost self explanatory:

    # urls.py
    urlpatterns = [
        path('admin/', admin.site.urls),
        path('transaction/<uuid:slug>/', TransactionDetailView.as_view()),
        path('transaction/<int:pk>/', TransactionDetailView.as_view()),
    ]

    # views.py
    class TransactionDetailView(DetailView):
        model = Transaction
        
    # models.py
    class Transaction(models.Model):
        slug = models.UUIDField(default=uuid.uuid4, db_index=True)
        amount = models.IntegerField(default=1000)

Generally speaking, class based views that accept a `pk` as parameter can accept instead a parameter 
named `slug`, and this slug parameter can be of any arbitrary type that you want (usually the UUID type).[[1]][1]

As you probably have noted I left the access though regular ID too, but just to empathize that django
accept both ways, you definitely don't want to keep both ways of access in your app. 


## Limiting the edit/update/delete to the object's owner.

One basic security question that you should address in your new application in the object level 
access permission, on a basic scale this usually revolves around only allowing the user to see, edit
and delete objects that he is the rightful owner. Along the years I have seem many suggestions on 
how to do that on CBV, they always felt strange and the reason is, while technically correct they 
are missing the fact that django have a simpler way of achieving that behavior.

Let's first say that we want to only display the Transactions that belong to the user:

    class TransactionListView(ListView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)

Straight forward, simple, elegant and well know... Now let's do the same for delete, hold your 
jaw:

    class TransactionDeleteView(DeleteView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)

And again for update:

    class TransactionUpdateView(UpdateView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)
            
And again for Detail:

    class TransactionDetailView(DetailView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)
            
That is it! You don't have to weirdly override the [dispatch][10], [get_object][11] or any other 
strange way you have found in the internet...

It took some time for me to realize that... I guess that whenever I saw the `get_queryset` in a view
for object level access I thought: _"That's weird, why there is a queryset method in an view that works
on a single object"_, but the fact is, this abstraction is converted to pure SQL and in SQL you don't
have a `get_object` or `get_queryset` it will always be a SELECT statement.

The `get_queryset` here is working as a pre-filter for the `get_object` and is ensuring that your 
actions only happens if the query is matching [[2]][2]. If this is still looking weird, I promess you that soon
this feeling will go away. And if it doesn't, at least the weirdness will be living in an abstraction
of the framework instead of in weird overrides that no one can agree about.


## Rendering variables from CBV methods.

This one I learned late, less than two year ago... I don't know if it's actually a relatively new 
feature from django or if it was just hidding in the documentation, but it was a game changer for me.

Whenever you need to expose some variables to templates the most common way is simply adding something 
to the `get_context_data`:


    class TransactionDeleteView(DeleteView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)

        def get_context_data(self, **kwargs):
            data = super().get_context_data(**kwargs)
            data['something_you_want_to_display_on_delete_template'] = '42'
            return data
 
There is a simpler way of doing that, a way that can help you have more organized code. Instead of 
packing a lot of variables and calls to the `get_context_data` you can create methods and access them
in templates, like this:

    class TransactionDeleteView(DeleteView):
        models = Transaction
        
        def get_queryset():
            return Transaction.objects.filter(user=self.request.user)

        def something_you_want_to_display_on_delete_template(self)
            return "42"

In your template:

    <h1> Are you sure about deleting {{ object }} </h1>
    <form>
        <buttom></button>
    </form> 
    <h2> Here I will display other relevant information to let you make a concious choice</h2>
    <p> {{ view.something_you_want_to_display_on_delete_template }} </p>


## Cache expensive computations that you need to access more than once per request.

Django have a [builtin decorator @cached_property][12] for caching short lived properties, the cache will live the same amount 
of time of the instance holding the property. As soon as the instance dies the cache dies. You often
need this when accessing the same resource twice in your template.

You can put the decorator at any level, it can be on a view property:

    # views.py
    class TransactionDetailView(DetailView):
        models = Transaction
        
        @cached_property
        def some_expensive_computation_from_view()
            return magic_that_will_return_42()

Or even better(?), it can save you some queries to database:
 
    # models.py
    class Transaction(models.Model):
        slug = models.UUIDField(default=uuid.uuid4, db_index=True)
        amount = models.IntegerField(default=1000)
        
        @cached_property    
        def some_expensive_queryset_from_model()
            return ExpensiveQueryset.filter().all()

To be honest, using it at Model level could get you in situations where you are going to reuse the method 
in other parts of your code and get cached when it shouldn't. Whenever you can, keep the `@cached_property`
at View level.

For completeness let's see a fake example of `@cached_property` in action, the first example will take 
4 seconds to render:

    # views.py
    class TransactionDetailView(DetailView):
        models = Transaction
        
        def correct_answer()
            time.sleep(2)
            return 42
            
    # transaction_detail.html
    {% if view.correct_answer == 42 %}
        {{ view.correct_answer }}
    {% endif %}

Now with the decorator it will take 2 seconds to render:

    # views.py
    class TransactionDetailView(DetailView):
        models = Transaction
        
        @cached_property
        def correct_answer()
            time.sleep(2)
            return 42
            
    # transaction_detail.html
    {% if view.correct_answer == 42 %}
        {{ view.correct_answer }}
    {% endif %}

Even with us calling the `view.correct_answer` twice in our template, it will actually be executed only
once during the conditional evaluation. The second call where we are printing the result will not 
happen, only the cached result will be returned.

## Final words

Each one of the tips presented here can carry some trade offs and particularities. While in some 
cases not using those features can be a conscious choice, trying to replicate some of them from 
scratch will probably lead you to a worse or at least more laborious code base.


[1]: https://docs.djangoproject.com/en/3.0/ref/class-based-views/mixins-single-object/#singleobjectmixin
[2]: https://docs.djangoproject.com/en/3.0/ref/class-based-views/mixins-single-object/#django.views.generic.detail.SingleObjectMixin.get_object
[10]: https://stackoverflow.com/questions/42649962/allow-only-the-author-of-the-post-to-edit-in-class-based-views-django
[11]: https://stackoverflow.com/questions/52673974/django-2-1-deleteview-only-owner-can-delete-or-redirect
[12]: https://docs.djangoproject.com/en/3.0/ref/utils/#django.utils.functional.cached_property