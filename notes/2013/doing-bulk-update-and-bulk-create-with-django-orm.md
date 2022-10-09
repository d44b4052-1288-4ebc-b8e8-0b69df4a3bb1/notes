---
[meta]
date = 2013-11-03
description = "How to proper use the django features to do bulk insertions and editions into database."
tags = "django, crud, orm, update, bulk, python, performance"
title = "Doing bulk update and bulk create with Django ORM"
---

It's not unusual the need to do bulk update/create in django applications, but if you don't use the right approach your views will increase the load time to unacceptable values.

Here is the common example where people starts out:

    # took 37 seconds
    def auto_transaction():
        for i in range(10000):
            name="String number %s" %i
            Record.objects.create(name=name)

Before django 1.4 we didn't have the built-in __bulk_create__, so the common way to do a bulk insertion was disabling the auto transaction in the beginning of operation and do it only once in the end:

    # took 2.65 seconds
    @transaction.commit_manually
    def manual_transaction():
        for i in range(10000):
            name="String number %s" %i
            Record.objects.create(name=name)
        transaction.commit()

But since Django 1.4 and the bulk_create built-in, insertions got a lot faster:

    # took 0.47 seconds
    def builtin():
        insert_list = []
        for i in range(10000):
            name="String number %s" %i
            insert_list.append(Record(name=name))
        Record.objects.bulk_create(insert_list)

### A similar effect happens to update operation ###
__The next examples are also executed against 10000 records__

Iterating over each record and manually updating the desired field:

    # took 72 seconds
    def auto_transaction():
        for record in Record.objects.all():
            record.name = "String without number"
            record.save()

Disabling transactions and committing only once in the end of process:


    # took 17 seconds
    @transaction.commit_manually
    def manual_transaction():
        for record in Record.objects.all():
            record.name = "String without number"
            record.save()
        transaction.commit()

Update using the django built-in update()

    # took 0.03 seconds
    def builtin():
        Record.objects.all().update(name="String without number");

__WARNING!__ Each method described here has it own particularity, and may not fit for your use case. Before you try any of them you should read the docs.

  - [https://docs.djangoproject.com/en/1.5/topics/db/transactions/](https://docs.djangoproject.com/en/1.5/topics/db/transactions/)
  - [https://docs.djangoproject.com/en/dev/topics/db/queries/#updating-multiple-objects-at-once](https://docs.djangoproject.com/en/dev/topics/db/queries/#updating-multiple-objects-at-once)
  - [https://docs.djangoproject.com/en/dev/ref/models/querysets/#bulk-create](https://docs.djangoproject.com/en/dev/ref/models/querysets/#bulk-create)
