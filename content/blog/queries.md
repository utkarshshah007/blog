+++ 
draft = false 
comments = false 
slug = "django-queries" 
tags = ["django", "python", "programming"]
categories = ["blog"]

title = "Exploring Relationship Queries with Django"
description = "Django's QuerySet feature makes it easy to surface related data efficiently."
date =  2018-05-02

showpagemeta = true
showcomments = true
+++

When creating a Django application, almost all valuable data models take advantage of relations. Often, as projects grow, we often need to make complex queries on relationships. To illustrate this, let's use two basic models as examples: `Tour` and `Concert`.

```python
class Tour(models.Model):
    name = models.CharField(max_length=200)


class Concert(models.Model):
	name = = models.CharField(max_length=200)
    tour = models.ForeignKey(Tour, on_delete=models.CASCADE)
    start = models.DateTimeField()
    tix_available = models.BooleanField()
```

This sets up a one-to-many relationship from `Tour` to `Concert` and adds some useful information to each `Concert` - when the show happens and how many tickets are still available.

### Simple Queries

While the extra information in `Concert` isn't a property of a `Tour` object, one can easily imagine a `TourDetailView` which needs to include information abut the next concert in the series and how many tickets are available for it.

```python
class TourDetailView(generic.DetailView):

    model = Tour

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)

        # Add in info on the next concert
        now = datetime.now()
        next = self.object.concert_set.filter(start__gt=now).earliest("start")
        context['next_concert'] = next
        return context
```

While this works just fine, it creates a one-time context object in the `View` that can't be reused elsewhere. Since this is information we'd want every time we use the `Tour` model, it would be much better if the logic for this lived in the model itself.

So how can we add a field that represents related 'Concert' data onto the 'Tour' without duplicating data in our database?

### Expanding the Model

The first, simplest answer is to create a `@property` method:

```python
class Tour(models.Model):
    name = models.CharField(max_length=200)

    @property
    def next_concert(self):
    	now = datetime.now()
        return self.concert_set.filter(start__gt=now).earliest("start")
```

This is allows us to easily get the information we want out of our `Tour` object:

```python
>>> t = Tour.objects.get(id=1)
>>> t
<Tour: 'Voodoo Lounge Tour'>
>>> t.next_concert
<Concert: {'name': 'Buenos Aires', 'tour': 1, 'start': '1995-02-09', 'tix_available': True}>
```

Seems simple enough!

But what happens when we want to make a `TourListView`?

```python
>>> tours = Tour.objects.all()
>>> tours.order_by('-next_concert__start')
Traceback (most recent call last):
  ...
django.core.exceptions.FieldError: Cannot resolve keyword 'next_concert' into field.
>>> 
```

Since `next_concert` isn't a model field, we can't use it in our `QuerySet` - that's because `QuerySet` accesses the underlying database directly - it doesn't know anything about our `@property` method.

We could simply pull each `Tour` object out in a `list()`, but that could mean a lot of unneccessary queries.

###  Taking advantage of `Annotate`

So how can we query this critical information in an efficient way? After we dig through Django's documentation and discover the `annotate` method, we can solve this too!

```python
>>> from django.db.models import Min
>>> tours = Tour.objects.annotate(
		next_concert_start=Min(
				'concert_set__start', 
				filter=Q(concert_set__start__gt=now)
			)
	)
>>> tours.values('next_concert_start').first()
{'next_concert_start': datetime.datetime(1995, 2, 9, 18, 9, 40, tzinfo=<UTC>)}
```

The annotate method allows us to attach an aggregate value to each of our `Tour` objects. This lets us use a single query to gather the data we need efficently.

*"But wait!" you say, "that's just the date! Don't we need to know if tickets are even available? How can we get access to the relevant Concert object itself?"*

###  Taking advantage of `Subquery`

That's a good point! Unfortunately, `annotate` won't be enough to help us there. `tix_available` for the next concert isn't something we can surface with an aggregation. Instead, we'll use `Subquery` for this.

Let's see if our `TourListView` can tell us how many tours have sold out their upcoming concerts.

```python
from django.db.models import OuterRef, Subquery

class TourListView(generic.ListView):

    model = Tour

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)

        tours = Tour.objects.all()

        # Use subquery to get ticket availability info
        now = datetime.now()
        concerts = Concert.objects.filter(tour=OuterRef('pk'))
        upcoming = concerts.filter(start__gt=now).order_by('-start')
        tours = tours.annotate(
        	tix_status=Subquery(upcoming.values('tix_available')[:1])
        )

        context['sold_out'] = tours.filter(tix_status=False).count()

        return context
```

`Subquery` allows us to nest queries so that we can ask these more complex questions. This feature allow us to convert what would normally have been several executions and database calls into a single query.

