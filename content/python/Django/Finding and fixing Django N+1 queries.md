## What are N+1 queries?

N+1 queries are when you unwittingly query the database in a for-loop and should be avoided so you don't DOS your database.

To illustrate the N+1 issue, let's start with a classic blog site model.
```python title=models.py
class Author(models.Model):
  name = models.TextField()

class Post(models.Model):
  name = models.CharField(max_length=100)
  content = models.TextField()
  author = models.ForeignKey(Author, on_delete=models.CASCADE)
```
Note the one-to-many relationship. One `Author` has many `Posts`.

The problem occurs when we access data from a related model in a loop, where each time we access the related field we make another call to the database.
```python
posts = Post.objects.all() # No query.

for post in posts: # Query database for posts
  print(post.author.name) # Query author
```
Here we make one query to get all the posts and N queries to get the author.name. Hence N+1 queries. 

## When does this happen?

Now you might say, well duh, this is obvious and who would make database queries in a for-loop? 

Django. Django would.

To be fair, it's not really Django's fault. It's our fault for not thinking about how the database is queried when designing our views. Django makes database queries in a for-loop  when your view uses a related field on a queryset.  This isn't the only time but this happens to me whenever I have a nested serializer, or serializer with a related field, when using the Django Rest Framework.

Consider this example where we're nesting a list of `Posts` inside the `Author` JSON in our API response.

```python
class PostSerializer(serializers.Serializer):
    name = serializers.CharField()
	content = serializer.CharField()

class AuthorSerializer(serializers.Serializer):
    name = serializers.CharField()
	posts = PostSerializer(many=True, read_only=True)
```

Given a Rest-style api layout, running `GET /api/v1/authors` using these serializers would make N+1 queries for _every_ author in the database.  Oops.

## The fix

Django Rest Framework makes a note of the fact that it's the [programmer's responsibility](https://www.django-rest-framework.org/api-guide/relations/#serializer-relations) to not make this mistake as fixing it in Django would be "too much magic". Fair enough. The docs also provides two solutions to the problem:
- [`select_related`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#select-related)
- [`prefetch_related`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#prefetch-related)

`select_related` makes a query with joins to avoid N+1 queries but doesn't work for many-to-many relationships.

Using `select_related`, we can fix our previous code.
```diff
-   posts = Post.objects.all() # No query.
+   posts = Post.objects.select_related("author").all() # No query

	for post in posts: # evaluate posts. Make a single query using joins.
	  print(post.author.name) # no more calls to db
```

Now when `posts` is accessed and the database is queried, Django makes a query with joins instead of multiple queries. Every time `post.author.name` is accessed, the cached queryset is used rather than making another call to the database.

[`prefetch_related`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#prefetch-related) works in a similar way but queries each table and does the joins in python.  In general, it's better do joins in the database and not in python but `prefetch_selected` also works for many-to-many and [`GenericRelations`](https://docs.djangoproject.com/en/5.1/ref/contrib/contenttypes/#django.contrib.contenttypes.fields.GenericRelation) and `select_related` does not.

## Summary
- N+1 queries are when we iteratively make calls to the database.
- N+1 queries occur because we're accessing columns in a related table but Django's ORM doesn't automatically sort this for us.
- Remember to use `select_related` or `prefetch_selected` when accessing many instances of related models to mitigate the issue.