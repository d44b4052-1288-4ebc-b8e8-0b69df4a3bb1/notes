---
[meta]
title = "Representing hierarchical data with django-rest-framework"
date = 2016-03-09
tags = ["django", "python", "django rest framework", "serializer", "tree", "drf"]
description = "Sometimes you want to provide features that require the manipulation and/or representation of hierarchical data, maybe it's a comment system or a product categorization feature, and if you don't have many constraints a design called adjacency list is often used."
---
Sometimes you want to provide features that require the manipulation and/or representation of hierarchical data,
 maybe it's a comment system or a product categorization feature, and if you don't have many constraints a design called 
 adjacency list is often used. It's so simple and natural that you probably have done something similar a few times, even
 not knowing the concept.

<pre><code class="python">
class Comment(models.Model):
    comment = models.CharField(max_length=250)
    parent = models.ForeignKey('self', related_name='reply_set', null=True)
    date = models.DateField(auto_now=True)
</code></pre>

In a simple adjacency list model, each record contains a reference to it predecessor, when this attribute is set to null,
 the record become a root node. The remaining records can also be root nodes or children's.
 
 With the simple model above, one can display comments in a tree format and basic operations will be
 simple and cheap: 
      
    #knowing who is the father is just a matter of consulting the model attribute:
    comment = Comments.objects.get(pk=1)
    comment.parent

    #accessing replies (children's)
    comment.reply_set.all()

But in order to explore the entire lineage of a conversation and provide the many features of a tree data 
structure, you may have to define some iteration/recursive methods that in practice can spawn many SQL consults and 
get uncomfortably slow for large amount of data. 

![django-rest-framework browsable api ;)](http://voorloopnul.com/files/2015/tree_data_drf.png?style=centerme "django-rest-framework browsable api")


The tree representation seen above is done using a nested serializer in [django-rest-framework](http://www.django-rest-framework.org/). 
Unfortunately it falls in the expensiveness trap, it walk thought the data recursively doing multiple sql consults in order to 
build the tree. On the bright side, it can be easily achieved with the following code:
  

**serializers.py**


    class RecursiveSerializer(serializers.Serializer):
        def to_representation(self, value):
            serializer = self.parent.parent.__class__(value, context=self.context)
            return serializer.data

    class CommentSerializer(serializers.ModelSerializer):
        reply_set = RecursiveSerializer(many=True, read_only=True)

        class Meta:
            model = Comment
            fields = ('id', 'date', 'comment', 'parent', 'reply_set')

I suggest that the data representation have an extra route to only display trees for root nodes. You should not direct 
filter in the ViewSet queryset attribute or you will lose the ability to update/delete children's using this endpoint. 
If you don't like this solution, you can instead implement a url-based filter with [django-filter](http://www.django-rest-framework.org/api-guide/filtering/#djangofilterbackend) or even override the 
ModelViewSet ```list``` method 

**views.py**


    class CommentViewSet(viewsets.ModelViewSet):
        # queryset = Comment.objects.filter(parent=None) # Don't
        queryset = Comment.objects.all()

        serializer_class = CommentSerializer

        @list_route()
        def roots(self, request):
            queryset = Comment.objects.filter(parent=None)
            serializer = self.get_serializer(queryset, many=True)
            return Response(serializer.data)

One important thing to account for is the delete behavior, [by default foreign keys uses CASCADE](https://docs.djangoproject.com/en/1.8/ref/models/fields/#django.db.models.ForeignKey.on_delete), 
so if you remove a higher level comment, all the replies will be ripped off too. To avoid this, you can change the parent 
field to uses ```on_delete=models.PROTECT``` or any of the other options. In the case of a comment system you can replicate
the reddit approach, change the comment content to something like *"Comment removed"*, remove *author information*
and keep the record in place to preserve the tree:


![Alt text](http://voorloopnul.com/files/2015/reddit_deleted.png?style=centerme "How reddit handle deleted comments")
