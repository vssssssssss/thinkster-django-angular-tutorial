# Extending Django's built-in User model
Django's built-in `User` model accomplishes a lot, but we don't live in a perfect world where built-in models give us everything we need. One shortcoming of the `User` model is that you can not extend it with new information. We wish to store extra information for each user, so this is a problem. The answer, it turns out, is to extend the `User` model by creating a second model with a one-to-one relationship with `User`. Creating this new model, `UserProfile`, is the focus of this section.

In Django, we use the concept of an "app" to organize our code in a meaningful way. Put simply, an app is a module that houses code for related models, views, serializers, etc. For example, an app called `authentication` might hold views for logging in, logging out and registering as well as a serializer for the `User` model. Each of these things is related to the authentication process in some way, so we include them in the `authentication` app.

Before we can create the `UserProfile` model, we need an application to store it in. As it happens, our first app will be called `authentication`.

Make a new app called `authentication` by running the following command:

    $ python manage.py startapp authentication

{x: startapp_authentication}
Make a Django app named `authentication`

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
Make a new model in `authentication/models.py` called `UserProfile`

Let's take a closer look at each attribute and method in turn.

    user = models.OneToOneField(User, primary_key=True, related_name='profile')

This is the one-to-one relationship with Django's `User` model that we talked about earlier. Having this relationship let's us access things like the user's name and email address. Notice that we set the `primary_key` argument to `True`. This ensures that the `pk` attribute of a `User` and the associate `UserProfile` are the same. We also pass the `related_name` key as `profile` which lets us say `User.profile` instead of `User.user_profile`, which is redundant.

*NOTE: Because we are using the associated `User` as the primary key, it is important to note that `UserProfile` objects **do not** have an `id` attribute. Instead, use `pk` in place of `id`. This is considered a best practice for Django in general, but it is especially important in this case.*

    tagline = models.CharField(max_length=140, blank=True, null=True)

The `tagline` attribute is a string that we will display on the user's profile page once we get that far. We arbitrarily set the `maximum_length` of this field to 140 characters.

*NOTE: The default value of `maximum_length` is `None`. This can cause a problem when your project must be portable to multiple databases. For more information, see the [CharField](https://docs.djangoproject.com/en/dev/ref/models/fields/#charfield) docs.*

    created_at = models.DateTimeField(auto_now_add=True)

This field records the time that the `UserProfile` object was created. By passing `auto_now_add=True` to `models.DateTimeField`, we are telling Django that this field should be automatically set when the object is created and non-editable after that.

    updated_at = models.DateTimeField(auto_now=True)

Similar to `created_at`, `updated_at` is automatically set by Django. The difference between `auto_now_add=True` and `auto_now=True` is that `auto_now=True` causes the field to update each time the object is saved.
 
    def __unicode__(self):
        return self.user.username

The `self.__unicode__()` method displays a string representation of the model when using the shell. By default, this is the name of the model (`UserProfile` here). We return the user's username instead to make it easier to distinguish users from one another.

    @receiver(post_save, sender=User)
    def create_profile_for_user(sender, instance=None, created=False, **kwargs):
        if created:
            UserProfile.objects.get_or_create(user=instance)

When a user registers, we will create a `User` model for them, but not a related `UserProfile` model. The `@receiver` decorator above `UserProfile.create_profile_for_user` solves this problem. Any time a `User` model is created, `UserProfile` is notified and creates a corresponding instance of itself. This ensures that we never have any orphan `User`s. That is to say, no `User`s without an associated `UserProfile`.

*NOTE: This concept is known as signal dispatching. Read [Signals](https://docs.djangoproject.com/en/1.7/topics/signals/) for more information.*

    @receiver(pre_delete, sender=User)
    def delete_profile_for_user(sender, instance=None, **kwargs):
        if instance:
            user_profile = UserProfile.objects.get(user=instance)
            user_profile.delete()

This is the complement to `UserProfile.create_profile_for_user`. When a `User` object is deleted, we want to delete the `UserProfile` object related to it. 

## Installing your first app
In Django, you must explicitly declare which apps are being used. Since we haven't added our `authentication` app to the list of installed apps yet, we will do that now.

Open `thinkster_django_angular_boilerplate/settings.py` and append `'authentication',` to `INSTALLED_APPS` like so:

    INSTALLED_APPS = (
        ...,
        'authentication',
    )

{x: install_authentication_app}
Install the `authentication` app

## Migrating your first app
When Django 1.7 was released, it  was like Christmas in September! Migrations had finally arrived!

Anyone with a background in Rails will find the concept of migrations familiar. In short, migrations handle the SQL needed to update the schema of our database so you don't have to. By way of example, consider the `UserProfile` model we just created. These models need to be stored in the database, but our database doesn't have a table for `UserProfile` objects yet. What do we do? We create our first migration! The migration will handle adding the tables to the database and offer us a way to rollback the changes if we make a mistake.

When you're ready, generate the migrations for the `authentication` app and apply them:

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

*NOTE: From now on, the output from migration commands will not be included for brevity.*

{x: initial_migration}
Generate the migrations for the `authentication` app and apply them

## Making yourself a superuser
One level of abstraction handled by Django's `User` model is the idea of roles. Different users have different levels of access in a given application. Some users are admins, some are regular users, some users may be part of a special testing group, etc.

In Django, a super user is the level of access a `User` can have. Because we want the ability to work with all features of our application, we will make ourselves a superuser account.

Making superusers is done from the terminal. After running the following command, Django will prompt you for some information and create a `User` with superuser access. Go ahead and give it a try.

    $ python manage.py createsuperuser

{x: create_superuser}
Make a new `User` with superadmin access

## Checkpoint
To make sure the receiver we set up before -- the one that creates a `UserProfile` when a `User` is created -- is working properly, let's jump into Django's shell:

    $ python manage.py shell

You should now see a new prompt: `>>>`. Inside the shell, we can get the `User` we just created like so:

    >>> from django.contrib.auth.models import User
    >>> u = User.objects.get(pk=1)
 
*NOTE: If you have created more than one `User`, you may want a different value for the `pk` argument.*

From here, you can confirm the receiver is working by accessing `u.profile`:

    >>> u.profile
    <UserProfile: james>

{x: get_user_profile}
Access the `UserProfile` associated with the `User` you just created

Now that we have verified `UserProfile` objects are created automatically, we can move on to serialization.
