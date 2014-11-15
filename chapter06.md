# Making a Post model
In this section we will make a new app and create a `Post` model similar to a status on Facebook or a tweet on Twitter. After we create our model we will move on to serializing `Post`s and then we will create a few new endpoints for our API.

## Making a posts app
First things first: go ahead and create a new app called `posts`.

    $ python manage.py start app posts

{x: django_app_posts}
Make a new app named `posts`

Remember: whenever you create a new app you have to add it to the `INSTALLED_APPS` setting. Open `thinkster_django_angular_boilerplate/settings.py` and modify it like so:

    INSTALLED_APPS = (
        # ...
        'posts',
    )

## Making the Post model
After you create the `posts` app Django made a new file called `posts/models.py`. Go ahead and open it up and add the following:

    from django.db import models
    from django.db.models.signals import pre_delete
    from django.dispatch import receiver

    from authentication.models import UserProfile


    class Post(models.Model):
        author = models.ForeignKey(UserProfile)
        content = models.TextField()

        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)

        def __unicode__(self):
            return '{0}'.format(self.content)

        @receiver(pre_delete, sender=UserProfile)
        def delete_thoughts_for_profile(sender, instance=None, **kwargs):
            if instance:
                posts = Post.objects.filter(author=instance)
                posts.delete()

{x: django_model_post}
Make a new model called `Post` in `posts/models.py`

Our method of walking through the code line-by-line is working well so far. Why mess with a good thing? Let's do it.

    author = models.ForeignKey(UserProfile)

When we created the `UserProfile` model we associated each `UserProfile` with a `User`. This is called a one-to-one relationship. Because each user can have a number of `Post`s, we want to set up a different kind of relationship: a many-to-one.

The way to do this in Django is with using a `ForeignKey` field to associate each `Post` with a `UserProfile`. 

Django is smart enough to know the foreign key we've set up here should be reversible. That is to say, given a `UserProfile`, you should be able to access that user's `Post`s. In Django these `Post` objects can be accessed through `UserProfile.post_set` (not `UserProfile.posts`).

    receiver(pre_delete, sender=UserProfile)
    def delete_thoughts_for_profile(sender, instance=None, **kwargs):
        if instance:
            posts = Post.objects.filter(author=instance)
            posts.delete()

This `pre_delete` hook is similar to the one we set up for `UserProfile`. The difference this time is that we delete the `Post` objects associated with a `UserProfile` before the `UserProfile` is deleted.

It's interesting to note that this creates a sort of chain. Before a `User` is deleted, the associated `UserProfile` is deleted. Before a `UserProfile` is deleted, the associated `Post` objects are deleted.

Now that the model exists, don't forget to migrate.

    $ python manage.py makemigrations
    $ python manage.py migrate

{x: django_model_post_migrate}
Make migrations for `Post` and apply them

## Serializing the Post model
Create a new file in `posts/` called `serializers.py` and add the following:

    from rest_framework import serializers

    from authentication.serializers import UserProfileSerializer
    from posts.models import Post


    class PostSerializer(serializers.ModelSerializer):
        author = UserProfileSerializer(required=False)

        class Meta:
            model = Post

            fields = ('id', 'author', 'content', 'created_at', 'updated_at')
            read_only_fields = ('id', 'created_at', 'updated_at')

        def get_validation_exclusions(self, *args, **kwargs):
            exclusions = super(PostSerializer, self).get_validation_exclusions()

            return exclusions + ['author']

{x: django_serializer_postserializer}
Make a new serializer called `PostSerializer` in `posts/serializers.py`

There isn't much here that's new, but there is one line in particular I want to look at.

    author = UserProfileSerializer(required=False)

We explicitly defined a number of fields in our `UserProfileSerializer` from before, but this definition is a little different.

When serializing a `Post` object, we want to include all of the author's information. Within Django REST Framework, this is known as a nested relationship. Basically, we are serializing the `UserProfile` related to this `Post` and including it in our JSON.

We pass `required=False` here because we will set the author of this post automatically.

    def get_validation_exclusions(self, *args, **kwargs):
        exclusions = super(PostSerializer, self).get_validation_exclusions()

        return exclusions + ['author']

For the same reason we use `required=False`, we must also add `author` to the list of validations we wish to skip.

## Making API views for Post objects
The next step in creating `Post` objects is adding an API endpoint that will handle performing actions on the `Post` model such as create or update.

Replace the contents of `posts/views.py` with the following:

    from rest_framework import generics, permissions

    from authentication.models import UserProfile
    from posts.models import Post
    from posts.permissions import IsAuthenticatedAndOwnsObject
    from posts.serializers import PostSerializer


    class PostListCreateView(generics.ListCreateAPIView):
        queryset = Post.objects.order_by('-created_at')
        serializer_class = PostSerializer

        def get_permissions(self):
            if self.request.method == 'POST':
                return (permissions.IsAuthenticated(),)
            return (permissions.AllowAny(),)

        def pre_save(self, obj):
            obj.author = UserProfile.objects.get(user=self.request.user)
            return super(PostListCreateView, self).pre_save(obj)


    class PostRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
        queryset = Post.objects.all()
        serializer_class = PostSerializer

        def get_permissions(self):
            if self.request.method not in permissions.SAFE_METHODS:
                return (IsAuthenticatedAndOwnsObject(),)
            return (permissions.AllowAny(),)

    class UserPostsListView(generics.ListAPIView):
        serializer_class = PostSerializer

        def get_queryset(self):
            return Post.objects.filter(
                author__user__username=self.kwargs['username']
            )

{x: django_view_post_listcreate}
Make a view for listing and creating `Post` objects

{x: django_view_post_retrieveupdatedestroy}
Make a view for reading, updating, and destroying `Post` objects

{x: django_view_user_post_list}
Make a view for listing `Post` objects by username

Do these views look similar? They aren't that different than the ones we made to create `User` objects.

    def pre_save(self, obj):
        obj.author = UserProfile.objects.get(user=self.request.user)
        return super(PostListCreateView, self).pre_save(obj)

`pre_save` is a method provided by the mixins used in `generics.ListCreateAPIView` (which we inherit from) that is called before an object is saved and that can be overridden. 

When a `Post` object is created it has to be associated with an author. Making the author type in their own username or id when creating adding a thought to the site would be a bad experience, so we handle this association for them with the `pre_save` hook. We simply grab the user associated with this request and make them the author of this `Post`.

    def get_permissions(self):
        if self.request.method == 'POST':
            return (permissions.IsAuthenticated(),)
        return (permissions.AllowAny(),)

Only authenticated users should be allowed to create new `Post`s. To satisfy this constraint we override `get_permissions`. 

If the request is a `POST` (the user is trying to create a new `Post` object), then we only allow authenticated users to continue. However, if the request is a `GET` (the only other HTTP verb this view accepts), then we will allow both authenticated and non-authenticated users to continue.

    class PostRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):

Even though we create this view and add an endpoint for it, we will not be using it in our application. It is considered good practice to make sure the software interface you implement are complete. For us, that makes being able to create, read, update, destroy, and list `Post` objects. 

We include this class to ensure we are providing a complete interface for handling `Post` objects.

    def get_permissions(self):
        if self.request.method not in permissions.SAFE_METHODS:
            return (IsAuthenticatedAndOwnsObject(),)
        return (permissions.AllowAny(),)

Similar to the other view, we need to handle the permissions for updating and destroying `Post`s here.

Django REST Framework provides a list of "safe" HTTP methods -- actions that do not modify data on the server, like `GET`. If the method for this request is not in that list, then we need to satisfy two constraints: the user must be authenticated and they must be own the object they are trying to modify.

`IsAuthenticatedAndOwnsObject` is a permission that we will implement ourselves now.

## Making the IsAuthenticationAndOwnsObject permission
Create `permissions.py` in the `posts/` directory with the following content:

    from rest_framework import permissions

    from posts.models import Post


    class IsAuthenticatedAndOwnsObject(permissions.BasePermission):
        def has_permission(self, request, view):
            if not request.user.is_authenticated():
                return False

            _id = self.kwargs['pk']

            return Post.objects.filter(id=_id, author_id=request.user).exists()

{x: django_permission_isauthenticatedandownsobject}
Make a new permission called `IsAuthenticatedAndOwnsObject` in `posts/permissions.py`

Let's step thought the code.

    class IsAuthenticatedAndOwnsObject(permissions.BasePermission):

When creating custom permissions with Django REST Framework, you should inherit from `permissions.BasePermission`. Django REST Framework, once again, handles a lot of boilerplate for us.

Custom permissions should return `True` if the user has permission to continue and `False` otherwise.

    if not request.user.is_authenticated():
        return False

Remember our first constraint? The user must be authenticated. `is_authenticated()` is a method provided by Django for handling this check.

    _id = self.kwargs['pk']

Presumably the request includes the `id` of the `Post` we are modifying. We grab that now to make the next line shorter.

    return Post.objects.filter(id=_id, author_id=request.user).exists()

This line satisfies our second constraint: the user must own this object. 

What we're doing here is filtering all `Posts` for anything matching the id and user provided. The `exists()` method returns `True` if our query found any matches and `False` otherwise.

## Making an API endpoint for posts
With the views created, it's time to add the endpoints to our API.

Open `thinkster_django_angular_boilerplate/urls.py` and add the following urls:

    from posts.views import PostListCreateView, \
        PostRetrieveUpdateDestroyView, UserPostsListView

    urlpatterns = patterns(
        # ...,
        url(r'^api/v1/users/(?P<username>[a-zA-Z0-9_@+-]+)/posts/$',
            UserPostsListView.as_view(), name='profile-posts'),
        url(r'^api/v1/posts/$', PostListCreateView.as_view(), name='posts'),
        url(r'^api/v1/posts/(?P<pk>[0-9]+)/$',
            PostRetrieveUpdateDestroyView.as_view(), name='post'),
        # ...,
    )

{x: django_url_post_listcreate}
Make an API endpoint for the `PostListCreateView` view

{x: django_url_post_retrieveupdatedestroy}
Make an API endpoint for the `PostRetrieveUpdateDestroyView` view

## Checkpoint
At this point, feel free to open up your shell with `python manage.py shell` and play around with creating and serializing `Thought` objects.

    >>> from authentication.models import UserProfile
    >>> from posts.models import Post
    >>> from posts.serializers import PostSerializer
    >>> profile = UserProfile.objects.get(pk=1)
    >>> post = Post.objects.create(author=profile, content='I promise this is not Google Plus!')
    >>> serialized_post = PostSerializer(post)
    >>> serialized_post.data

{x: checkpoint_create_post}
Play around with the `Post` model and `PostSerializer` serializer in Django's shell

We will confirm the views are working at the end of the next section.
