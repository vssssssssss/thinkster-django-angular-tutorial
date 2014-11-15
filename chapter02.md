# Serializing the User and UserProfile Models
Our application will make AJAX requests to the server to get the data we intend to display. Before we can send that data back to the client, we need to format it in a way that the client can understand; in this case, we choose JSON. The process of transforming Django models to JSON is called serialization and that is what we will talk about now.

We need to write two serializers: one for the `User` model and one for the `UserProfile` model.  We will call these `UserSerializer` and `UserProfileSerializer`, respectively.

## Django REST Framework
As part of the boilerplate project you cloned earlier, we have included a project called Django REST Framework. Django REST Framework is a toolkit that provides a number of features common to most web applications, including serializers. We will make use of these features throughout the tutorial to save us both time and frustration. Our first look at Django REST Framework starts here.

## UserSerializer
Before we write our serializers, let's create a `serializers.py` file inside our `authentication` app:

    $ touch authentication/serializers.py

{x: create_serializers_module}
Create a `serializers.py` file inside the `authentication` app

Open `authentication/serializers.py` and add the following code and imports:

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
Make a serializer called `UserSerializer` in `authentication/serializers.py`

*NOTE: From here on, we will declare imports that are used in each snippet. These may already be present in the file. If so, they do not need to be added a second time.*

*NOTE: All imports should be at the top of the file.*

Let's take a closer look.

    class Meta:

The `Meta` sub-class defines metadata the serializer requires to operate. We have defined a few common attributes of the `Meta` class here.

    model = User

Because this serializers inherits from `serializers.ModelSerializer`, it should make sense that we must tell it which model to serialize. Specifying the model creates a guarantee that only attributes of that model or explicitly created fields can be serialized. We will cover serializing model attributes now and explicitly created fields shortly.

    fields = (
        'id', 'username', 'email', 'first_name', 'last_name', 'password'
    )

The `fields` attribute of the `Meta` class is where we specify which attributes of the `User` model should be serialized. We must be careful when specifying which fields to serialize because some fields, like `is_superuser`, should not be available to the client for security reasons.

    write_only_fields = ('password',)

Notice that we are telling our serializer to serialize the `password` attribute. This seems like a bad idea, don't you think? Django does not store passwords in plain text, but even the hashed and salted version of a user's password is not something the client should have access to.

We solve this problem by telling the serializer that the `password` attribute should only be writable. In other words, the serializer should only recognize `password` if we are updating a `User`.

    def restore_object(self, attrs, instance=None):

Earlier we mentioned that we sometimes want to turn JSON into a Python object. This is called deserialization and it is handled by the `self.restore_object()` method.

    user = super(UserSerializer, self).restore_object(attrs, instance)

Most of the boilerplate code for updating any object is handled by the default implementation of `self.restore_object()`. To save time, we delegate that work to the parent of `UserSerializer`. 

    password = attrs.get('password', None)

    if password:
                user.set_password(password)

Overwriting `self.restore_object()` is not required unless there is extra work that needs to be done. In the case of a `User`, we need to use the `User.set_password()` method to make sure the user's new password is set in a secure way.

If we didn't want to support updating passwords, we could leave out `self.restore_object()` completely.

## UserProfileSerializer
We just created a serializer for the `User` model. Now we need one for the `UserProfile` model. The serializers will be similar in nature, but `UserProfileSerializer` will present a few new concepts around handling model relationships (one-to-one) with serializers.

We need to add the following code to `authentication/serializers.py`:

    from rest_framework import serializers

    from authentication.models import UserProfile


    class UserProfileSerializer(serializers.ModelSerializer):
        id = serializers.IntegerField(source='pk', read_only=True)
        username = serializers.CharField(source='user.username', read_only=True)
        email = serializers.CharField(source='user.email', required=False)
        first_name = serializers.CharField(source='user.first_name', required=False)
        last_name = serializers.CharField(source='user.last_name', required=False)

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
Make a serializer called `UserProfileSerializer` in `authentication/serializers.py`

For the sake of brevity, we will skip a lot of the code and talk instead about the new ideas this serializer presents.

The first thing you may notice is that we define a number of fields at the top of the class. There are two reasons for this. 

    id = serializers.IntegerField(source='pk', read_only=True)

As mentioned in an earlier note, `UserProfile` does not have an `id` attribute because we made the associated `User` the primary key. Because it is standard to include an `id` key in our JSON, we explicitly creating an `id` field whose source is `UserProfile.pk`. In this case, `UserProfile.pk` is the `id` of the associated `User`.

    username = serializers.CharField(source='user.username', read_only=True)
    email = serializers.CharField(source='user.email', required=False)
    first_name = serializers.CharField(source='user.first_name', required=False)
    last_name = serializers.CharField(source='user.last_name', required=False)

When we serialize a `UserProfile` object, we want to include information about the related `User`. After all, the whole point of `UserProfile` is to *extend* `User`. Django REST Framework allows us to do this by specifying which fields of the related model we want to include. Here we have chosen `username`, `email`, `first_name` and `last_name`.

These are the explicitly defined fields I mentioned earlier. Each field accepts a `source` attribute whose value is `<related model>.<related model attribute>`. For example, if we want to include the `username` of the associated `User`, we set the source to `user.username`.

    read_only_fields = ('created_at', 'updated_at',)

Conversely to `write_only_fields` mentioned before, sometimes we have fields that should be read-only. In this case, both `created_at` and `updated_at` will update themselves, so we make them non-writable.

    user.email = attrs.get('user.email', user.email)
    user.first_name = attrs.get('user.first_name', user.first_name)
    user.last_name = attrs.get('user.last_name', user.last_name)

Later on, we will add support for updating a user's email, first name, and last name. When updating a related model, each attribute must be explicitly set.

    user.save()

When updating a related model, you must also explicitly save it.
