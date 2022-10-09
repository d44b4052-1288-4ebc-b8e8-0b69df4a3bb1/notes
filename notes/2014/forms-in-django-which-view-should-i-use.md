---
[meta]
title = "Forms in django, which view should I use?"
date = 2014-06-10
tags = ["django", "forms", "views", "crud"]
description = "Learn how to choose and use the correct features for exposing forms and views in django."
---
Django is a great framework, it has most features that you would need when building web applications, but sometimes it can be overwhelming. In this small article **I will show you five different** (yet close) ways to display forms and create objects from them. I hope that in the end you will be able to better understand how each way works and choose the best approach to use in your projects.

For completeness, at the end of this article you can download a django project including all the five views plus their forms and models.

### Function based view (FBV)

There are a few ways to display and interact with django forms, probably the most common way is using django functional views, they are simple, flexible and virtually everyone starts with them, here is an example:

    def example1view(request):
        form = Example1Form(request.POST or None)
        if form.is_valid():
            data = form.cleaned_data
            star = Star(**data)
            star.save()
            return HttpResponseRedirect(reverse('success'))
        return render(request, 'template.html', {'form': form})

They are a good way to work with forms that don't require a django model, but for forms tied to models it bring some boilerplate to your code. For just one or two models it's OK, but in crud-intensive applications, it will make things less fun.

### Function based views with ModelForms

Alternatively, if using ModelForms, a few lines can be omitted (Although the real gain is because you don't have to manually define the form):

    def example2view(request):
        form = Example2Form(request.POST or None)
        if form.is_valid():
            form.save()
            return HttpResponseRedirect(reverse('success'))
        return render(request, 'template.html', {'form': form})
        
For the sake of simplicity each new example will be using ModelForms, if you don´t know how to customise them, drop a line in the comments and I will consider an article about it.


### Class based views (CBV)

Another alternative you have is the class-based views, it preserves the balance of flexibility and simplicity, plus, makes the code more clear to read and understand (right after you learn them), but also can make things longer.

    class Example3View(View):
        form_class = ModelFormExample
        template_name = 'template.html'
    
        def get(self, request, *args, **kwargs):
            form = self.form_class()
            return render(request, self.template_name, {'form': form})
    
        def post(self, request, *args, **kwargs):
            form = self.form_class(request.POST)
            if form.is_valid():
                form.save()
                return HttpResponseRedirect(reverse('success'))
            return render(request, self.template_name, {'form': form})

In class based views you have specific places to process get/post requests, and can organise variables in the class scope.  

### Class based views with Generic Views

But class based views can be made even easier by inheriting generic views like [FormView](http://ccbv.co.uk/projects/Django/1.6/django.views.generic.edit/FormView/). This one is specially useful for the cases where your form is not tied to django models. But even for model related operations it has a strong appeal.

    class Example4View(FormView):
        template_name = 'template.html'
        form_class = ModelFormExample
    
        def form_valid(self, form):
            form.save()
            return super(Example4View, self).form_valid(form)
    
        def get_success_url(self):
            return reverse('success')

### Generic CBV for CRUD operations

And finally, there is the [CreateView](http://ccbv.co.uk/projects/Django/1.6/django.views.generic.edit/CreateView/), a generic CBV designed for insertion operations, they don't require neither a form nor modelform, they are easy to understand, short, can save you a lot of time and eventually you will learn how to extend them when in need of extra features:

    class Example5View(CreateView):
        model = Star
        template_name = 'template.html'
        success_url = reverse_lazy('success')
            
At this point you may be curious about which one you should use, the truth is: all of them! As the time goes you probably will ditch the FBV in favour of the CBV approach, but this is not a rule, there are plenty of people who don't use and don't like CBV, and it is OK! 

I particularly think that class based views can make the code more clear (but just for people who already know the CBV way), but sometimes I still use FBV.

