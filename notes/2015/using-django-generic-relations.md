---
[meta]
title = "Using django generic relations"
date = 2015-02-22
tags = ["django", "models", "generic relations", "orm", "relationship"]
description = "Using django generic relations to create generic features that can reference any model without an explicit relationship."
---
# Using django generic relations

Let's say that we are going to build an application for storing data about celestial bodies, we start creating a model for each body that we want to represent, and later we decide that our system must have the ability to save notes for each one of those bodies. If we have many bodies, adding a new model covering the **notes feature** for each model would be to much <strike>d</strike>**ry**: 

    from django.db import models
    
    class BaseEntity(models.Model):
        class Meta:
            abstract = True
    
    class Star(BaseEntity): 
        pass
    
    class Planet(BaseEntity):
        pass
    
    class Moon(BaseEntity):
        pass
    
    class StarNote(models.Model): 
        star = models.ForeignKey(Star)
        note = models.CharField(max_lenght=500)
    
    class PlanetNote(models.Model): 
        planet = models.ForeignKey(Planet)
        note = models.CharField(max_lenght=500)
    
    class MoonNote(models.Model): 
        moon = models.ForeignKey(Moon)  
        note = models.CharField(max_lenght=500)    

Generic relations - as the name implies - will let us relate generic objects of type **Star**, **Planet**, **Moon** *(or any other)*; to objects of type **Note**. It works using the content_type of **any object** and their pk as GenericForeignKey for a **Note**:

    from django.contrib.contenttypes import generic
    from django.contrib.contenttypes.models import ContentType
    from django.db import models
    
    class BaseEntity(models.Model):
        class Meta:
            abstract = True
    
        def get_content_type(self):
            return ContentType.objects.get_for_model(self).id
    
    class Star(BaseEntity): 
        pass
    
    class Planet(BaseEntity):
        pass
    
    class Moon(BaseEntity):
        pass
    
    class Note(models.Model):
        note = models.CharField(max_lenght=500)
        content_type = models.ForeignKey(ContentType)
        object_id = models.PositiveIntegerField()
        content_object = GenericForeignKey('content_type', 'object_id')

The benefits, are really straight: 

 * in real life we will end with less code.
 * most changes to our **note feature** will be applied only in one place.
 * any model in our application will be able to use the **note feature**.
 * we will benefit of loose coupling.
 
In practice, if we want to add a note to Sun, we do:


    > sun = Star.objects.get(pk=1)
    > new_note = Note(content_object=sun, note='This is our star...')
    > new_note.save()

or something like this:

    > sun = Star.objects.get(pk=1)
    > sun.get_content_type()
    > 23
    > new_note = Note(content_type=23, object_id=1 note='This is our star...')
    > new_note.save()

later, if we want to retrieve all notes for planet earth, we do:

    > earth = Planet.objects.get(pk=2)
    > Note.objects.filter(content_object=earth)
    > [Note-1, Note-2, Note-3]


Our example was a note system, but using this concept we can also implement features like **Attachments** and **Tags**. The original documentation has more details about using [generic relations](https://docs.djangoproject.com/en/1.7/ref/contrib/contenttypes/#generic-relations), and a particularly useful section covering [reverse generic relations](https://docs.djangoproject.com/en/1.7/ref/contrib/contenttypes/#reverse-generic-relations), that will let us do reverse lookups of objects like this:

    > earth = Planet.objects.get(pk=2)
    > earth.notes.all()
    > [Note-1, Note-2, Note-3]
