# Serializing the User and UserProfile Models
Our application will make AJAX requests to the server to get the data we intend to display. Before we can send that data back to the client, we need to format it in a way that the client can understand; in this case, we choose JSON. The process of transforming Django models to JSON is called serialization and that is what we will talk about now.

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

            password = attrs.get('password', None)

            if password:
                user.set_password(password)

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
