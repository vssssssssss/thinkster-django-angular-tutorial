# Extending Django's built-in User model
When it comes to modeling a user, Django's built-in `User` model goes a very long way. Everything from requiring certain information to validating that information to authenticating the user is already handled for us.

Unfortunately for us, we don't live in a perfect world where built-in models give us everything we need. Because we will be making AJAX requests for all of our data, we need to create a serializer for Django's `User` model at the very least.

A serializer takes a Python object and transforms it into something we can work with in the browser. In our case, this will be JSON. JSON has become a standard for transmitting data from the server to the client with asynchronous (AJAX) requests.

Django's built-in `User` model has one shortcoming: you can't outright extend it with new information<sup>1</sup>. For example, we will give our users a tagline that will show up in their profile, but the `User` model does not include a `tagline` attribute and we can't give it one.

There are a few ways to accomplish our goal<sup>2</sup>. In this tutorial, we choose to extend the `User` model with a new model of our own design that has a One-To-One relationship with a `User`. We will call this model `UserProfile`.

Before we can start creating models, we need to create a new Django app<sup>3</sup>. Our first app will be called `authentication` and will house code related to `User`s, `UserProfile`s, and views for logging in, logging out, and registration. 

Create the `authentication` app by navigating to the root of your Django project and running the following command:

    $ python manage.py startapp authentication

{x: startapp_authentication}
Create a Django app named `authentication`

## Creating the UserProfile model
To get started, we will create the `UserProfile` model we talked about earlier.

Open `authentication/models.py` in your favorite text editor and edit it to reflect the following:

    from django.contrib.auth.models import User
    from django.db import models
    from django.db.models.signals import post_save, pre_delete
    from django.dispatch import receiver


    class UserProfile(models.Model):
        user = models.OneToOneField(User, primary_key=True, related_name='profile')
        
        tagline = models.CharField(max_length=140, blank=True, null=True)
        
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
        
        def __unicode__(self):
            return self.user.username
            
        @receiver(post_save, sender=User)
        def create_profile_for_user(sender, instance=None, created=False, **kwargs):
            if created:
                UserProfile.objects.get_or_create(user=instance)
    
        @receiver(pre_delete, sender=User)
        def delete_profile_for_user(sender, instance=None, **kwargs):
            if instance:
                user_profile = UserProfile.objects.get(user=instance)
                user_profile.delete()

{x: create_user_profile_model}
Create a `UserProfile` model

Let's take a closer look at each attribute and method in turn.

    user = models.OneToOneField(User, primary_key=True, related_name='profile')

This is the One-To-One relationship with Django's `User` model that we talked about earlier. Having this relationship let's us access things like the user's name and email address. Notice that we set the `primary_key` argument to `True`. This ensures that the `pk` attribute of a `User` and the associate `UserProfile` are the same<sup>4</sup>. We also pass the `related_name` key as `profile` which lets us say `<user object>.profile` instead of `<user object>.user_profile`, which is redundant.

    tagline = models.CharField(max_length=140, blank=True, null=True)

The `tagline` attribute is a string that we will display on the user's profile page once we get that far. We arbitrarily set the `maximum_length` of this field to 140 characters<sup>5</sup>.

    created_at = models.DateTimeField(auto_now_add=True)

This field records the time that the `UserProfile` object was created. By passing `auto_now_add=True` to `models.DateTimeField`, we are telling Django that this field should be automatically set when the object is created and non-editable after that.

    updated_at = models.DateTimeField(auto_now=True)

Similar to `created_at`, `updated_at` is automatically set by Django. The difference between `auto_now_add=True` and `auto_now=True` is that `auto_now=True` causes the field to update each time the object is saved.
 
    def __unicode__(self):

The `self.__unicode__()` method displays a string representation of the model. By default, this is the name of the model (`UserProfile` here). We return the user's username instead.

    def create_profile_for_user(sender, instance=None, created=False, **kwargs):

When a user registers, we create a `User` model for them, but not a related `UserProfile` model. The `@receiver` decorator above `UserProfile.create_profile_for_user` solves this problem. Any time a `User` model is created, `UserProfile` is notified a creates a corresponding instance of itself. This ensures that we never have any open `User`s. That is to say, no `User`s without an associated `UserProfile`.

    def delete_profile_for_user(sender, instance=None, **kwargs):

This is the complement to `UserProfile.create_profile_for_user`. When a `User` object is deleted, we want to delete the `UserProfile` object related to it. 

That sums up the `UserProfile` model. 

In Django, you must explicitly declare which apps are being used. Since we haven't added our `authentication` app to the list of installed apps yet, we will do that now.

# Installing your first app
Open `thinkster_django_angular_boilerplate/settings.py` and append `'authentication',` to `INSTALLED_APPS` like so:

    INSTALLED_APPS = (
        ...,
        'authentication',
    )

{x: install_authentication_app}
Install the `authentication` app

Earlier we made a reference to a Django command for generating migrations -- `python manage.py makemigrations`. Now we will talk briefly about what migrations are, why they are important, and how to use them.

A Django Migration is a Python class that tells Django what changes need to be made to the database. For example, when you create a new model like we just did, you need to add a corresponding table to your database. Migrations make your life easier by automatically generating the SQL needed to create that table. Furthermore, migrations allow you to roll back changes if you make a mistake.

Let's go ahead and generate the migrations for our app and apply them:

    $ python manage.py makemigrations
    Migrations for 'authentication':
        0001_initial.py:
            - Create model UserProfile
    $ python manage.py migrate
    Operations to perform:
        Synchronize unmigrated apps: rest_framework
        Apply all migrations: admin, authentication, contenttypes, auth, sessions
    Synchronizing apps without migrations:
        Creating tables...
        Installing custom SQL...
        Installing indexes...
    Running migrations:
        Applying authentication.0001_initial... OK

*NOTE: From here on, the output from migration commands will not be included for brevity.*

{x: initial_migration}
Create your first migration and apply it

## Creating your first user
In Django, a superuser is the highest level of access a `User` model can have. Because we may way to test some features that require us to be an admin, we will make ourselves a superuser account.

The easiest way to do this is from your terminal with the following command:

    $ python manage.py createsuperuser

From here you will be asked for your username, email, and password and a `User` will be created for you with superuser access. Go ahead and try this now.

{x: create_superuser}
Create a superuser for yourself

## It's shell time
To make sure the receiver we set up before -- the one that creates a `UserProfile` when a `User` is created -- is working properly, let's jump into Django's shell:

    $ python manage.py shell

You should now see a new prompt: `>>>`. Inside the shell, we can get the `User` we just created like so:

    >>> from django.contrib.auth.models import User
    >>> u = User.objects.get(pk=1)
 
*NOTE: If you have created more than one `User`, you may want a different value for the `pk` argument.*

From here, all we have to do to confirm the receiver is working is access `u.profile`:

    >>> u.profile
    <UserProfile: james>

{x: get_user_profile}
Access the `UserProfile` associated with the `User` you just created

Now that we have verified `UserProfile` objects are created automatically, we can move on to serialization.

*NOTE: We will test the other receiver we created -- the one that deletes the `UserProfile` associated with a `User` -- later in the tutorial. For now, we just assume it works.*

## Serializing User models with Django REST Framework
Django models are great for -- you guessed it -- modeling data in Python. What they are not well suited for is displaying information on the client. To send data from our server to the client, we need to serialize the models into a format that our client understands. The popular format these days in JSON, so that is what we will use.

Luckily, the boilerplate project you started from has already installed a project called Django REST Framework. Django REST Framework makes it easy to serialize Django models into JSON, among other things, and we will use it extensively throughout this tutorial.

We need to write two serializers: `UserSerializer` and `UserProfileSerializer`. We won't be directly modifying `User` objects, but the `ModelSerializer` class in Django handles both serialization and deserialization. When we create and destroy `User`s, we will need to deserialize them.

Before we write our serializers, let's create a `serializers.py` file inside our `authentication` app. Run the following command from the root of your project:

    $ touch authentication/serializers.py

{x: create_serializers_module}
Create a `serializers.py` file inside the `authentication` app

Open `authentication/serializers.py` and add the following snippet:

    from django.contrib.auth.models import User

    from rest_framework import serializers


    class UserSerializer(serializers.ModelSerializer):
        class Meta:
            model = User
            fields = (
                'id', 'username', 'email', 'first_name', 'last_name', 'password'
            )
            write_only_fields = ('password',)

        def restore_object(self, attrs, instance=None):
            user = super(UserSerializer, self).restore_object(attrs, instance)

            if hasattr(attrs, 'password'):
                user.set_password(attrs.get('password'))

            return user

{x: create_user_serializer}
Create a `UserSerializer` class in `authentication/serializers.py`

Let's take a closer look.

    class Meta:

The `Meta` sub-class defines information the serializer needs to do it's job. We have defined a few common attributes of the `Meta` sub-class here.

    model = User

The serializer needs to know what model it will be serializing. Specifying the model creates a guarantee that only attributes of that model or explicitly created fields can be serialized. We will cover serializing model attributes now and explicitly created fields shortly.

    fields = (
        'id', 'username', 'email', 'first_name', 'last_name', 'password'
    )

The `fields` attribute of the `Meta` class is where we specify which attributes of the `User` model should be serialized. We must be careful when specifying which fields to serialize because some fields, like `is_superuser`, should not be available to the client for security reasons.

    write_only_fields = ('password',)

Notice that we are telling our serializer to include the `password` attribute. This seems like a bad idea, don't you think? Django does not store passwords in plain text, but even the hashed and salted version of a user's password is not something the client should have access to.

We solve this problem by telling the serializer that the `password` attribute should only be writable. In other words, the serializer should only recognize `password` if we are updating a `User`.

    def restore_object(self, attrs, instance=None):

Earlier we mentioned that we sometimes want to turn JSON into a Python object. This is called deserialization and it is handled by the `self.restore_object()` method.

    user = super(UserSerializer, self).restore_object(attrs, instance)

Most of the boilerplate code for updating any object is handled by the default implementation of `self.restore_object()`. To save time, we delegate that work to the parent of `UserSerializer`. 

    if hasattr(attrs, 'password'):
                user.set_password(attrs.get('password'))

Overwriting `self.restore_object()` is not required unless there is extra work that needs to be done. In the case of a `User`, we need to use the `User.set_password()` method to make sure the user's new password is set in a secure way.

If we didn't want to support updating your password, we could leave out `self.restore_object()` completely.

Moving on!

## Serializing UserProfile objects
We just created a serializer for the `User` model. Now we need one for the `UserProfile` model. The serializers will be similar in nature, but `UserProfileSerializer` will present a few new concepts around handling model relationships (One-to-One) with serializers.

We need to add import to `authentication/serializers.py`:

    from authentication.models import UserProfile

With `UserProfile` imported, add the following class to `authentication/serializers.py`:

    class UserProfileSerializer(serializers.ModelSerializer):
        id = serializers.IntegerField(source='pk', read_only=True)
        username = serializers.CharField(source='user.username', read_only=True)
        email = serializers.CharField(source='user.email')
        first_name = serializers.CharField(source='user.first_name')
        last_name = serializers.CharField(source='user.last_name')

        class Meta:
            model = UserProfile
            fields = (
                'id', 'username', 'email', 'first_name', 'last_name', 'tagline',
                'created_at', 'updated_at',
            )
            read_only_fields = ('created_at', 'updated_at',)

        def restore_object(self, attrs, instance=None):
            profile = super(UserProfileSerializer, self).restore_object(
                attrs, instance
            )

            if profile:
                user = profile.user

                user.email = attrs.get('user.email', user.email)
                user.first_name = attrs.get('user.first_name', user.first_name)
                user.last_name = attrs.get('user.last_name', user.last_name)

                user.save()

            return profile

{x: create_user_profile_serializer}
Create `UserProfileSerializer` in `authentication/serializers.py`

For the sake of brevity, we will skip a lot of the code and talk instead about the new ideas this serializer presents.

The first thing you may notice is that we define a number of fields at the top of the class. There are two reasons for this. 

    id = serializers.IntegerField(source='pk', read_only=True)

As mentioned in an earlier note, `UserProfile` does not have an `id` attribute because we made the associated `User` the primary key. Because it is standard to include an `id` key in our JSON, we explicitly creating an `id` field whose source is `UserProfile.pk`. In this case, `UserProfile.pk` is the `id` of the associated `User`.

    username = serializers.CharField(source='user.username', read_only=True)
    email = serializers.CharField(source='user.email')
    first_name = serializers.CharField(source='user.first_name')
    last_name = serializers.CharField(source='user.last_name')

We also want to include some information about the `User` object. The way to do this with Django REST Framework is to specify example what fields of the related model you want to include. Here we have chosen `username`, `email`, `first_name` and `last_name`.

These are the explicitly defined fields I mentioned earlier. Each field accepts a `source` attribute whose value is `<related model>.<related model attribute>`. For example, if we want to include the `username` of the associated `User`, we set the source to `user.username`.

    read_only_fields = ('created_at', 'updated_at',)

Conversely to `write_only_fields` mentioned before, sometimes we have fields that should be read-only. In this case, both `created_at` and `updated_at` will update themselves, so we make them non-writable.

    user.email = attrs.get('user.email', user.email)
    user.first_name = attrs.get('user.first_name', user.first_name)
    user.last_name = attrs.get('user.last_name', user.last_name)

Later on, we will add support for updating a user's email, first name, and last name. When updating a related model, each attribute must be explicitly set.

    user.save()

When updated a related model, you must also explicitly save it.

## Onwards to bigger and better things .. and some AngularJS!
You're my hero! Look how much you've already accomplished! In a short time, you've gotten a working Django project running, you extended the built-in `User` model to store more information about each user, and you have created serializers for both `User` and `UserProfile` so we can communicate them to our client.

Are you sufficiently exhausted yet? Good! Take a break.

When you come back from your break, move on to the next chapter where we build over the views for our authentication system and jump into some AngularJS to build login and registration forms.

## Chapter notes
*<sup>1</sup> This may seem annoying, but the design decisions behind this are solid. It's A Good Thing&#0153;.*

*<sup>2</sup> For a full, in-depth discussion on your choices for customizing the functionality of a `User`, see [Customizing authentication in Django](https://docs.djangoproject.com/en/1.7/topics/auth/customizing/).*

*<sup>3</sup> Django apps are not to be confused with Django projects. An app is simply a module of a larger project. For more information, see [Projects and Applications](https://docs.djangoproject.com/en/1.7/ref/applications/#projects-and-applications).*

*<sup>4</sup> Because we are using the associated `User` as the primary key, it is important to note that `UserProfile` objects **do not** have an `id` attribute. Instead, use `pk` in place of `id`. This is considered a best practice for Django in general, but it is especially important in this case.*

*<sup>5</sup> The default value of `maximum_length` is `None`. This can cause a problem when your project must be portable to multiple databases. For more information, see the [CharField](https://docs.djangoproject.com/en/dev/ref/models/fields/#charfield) docs.*
