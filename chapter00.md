# Welcome, Third of Five<sup>1</sup>!
In this tutorial you will learn how to build a web application with Django and AngularJS. By the end of the tutorial, you will have built a simplified Google+ clone (with a hint of Borg) and deployed it to Heroku. 

From there, we will give you some ideas on what features you can implement to extend the app. The hardest part of making software is actually building something start-to-finish and we will be with you every step of the way, ensuring your success. Have no fear, Thinkster.io is here!

Before we hit the books are learn how to build web applications with Django and AngularJS, let's take a moment to explore the motivations behind this tutorial. There are also a few things you need to know up front to get the best bang for your buck.

## What is the goal of this tutorial?
Here at Thinkster, our number one goal has always been to create high value, in depth content with a low barrier to entry. We release this content for free with the hope that you find it both helpful and informative.

Each tutorial has a specific goal. In this tutorial, that goal is to introduce you to Django, Django REST Framework, and AngularJS and how you can combine these technologies to build amazing web applications. 

Furthermore, there is an emphasis on building good engineering habits. This means doing things like actively considering the trade offs that come from making architectural decision, building good habits for testing your code<sup>2</sup>, and gaining an intuitive understanding of how your code works. While these things may not sound like fun, they are essential to becoming a well rounded software developer.

## Who is this tutorial for?
This is a question that anyone who creates content must answer, and it's a difficult question. Our goal is to make this tutorial useful for novices and experienced developers alike.

If this is your first rodeo, we have tried to be thorough and logical in our explanations. We set out with a goal of making this tutorial accessible to anyone with enough interest to take the time necessary to learn and understand the concepts presented. It is our belief that we have accomplished this goal.

For those of you who have been around the block a few times, we know you don't need the basics explain to you. We have tried our best to make this tutorial easy to skim so you can make use of your existing knowledge to speed up the process of reading the tutorial and identify where unfamiliar concepts are presented with ease so you can grok them and move on. 

With that said, if you are an experienced AngularJS and Django developer, you probably won't find anything new here. We invite you to make your way through the tutorial regardless, but with the understanding that you probably know just as much (if not more) than we do.

## What are we going to build?
Throughout this tutorial, we will be building an over-simplified Google+ clone for the Borg (see the next section for more information on the Borg) with features like authentication and posting status updates.

A good project for learning purposes is much like a good interview question. The initial goals tend to be simple and straightforward while allowing for the difficulty to progress naturally. We believe that a social network like Google+ accomplishes this goal.

In the first release of the tutorial, we will include the basics of authentication (login, logout, and creating and deleting accounts) and updating your status. As time goes on and we gather feedback on what you want to see, we will expand the tutorial. Potential features that we could add include communities, voting (Like, +1, etc.), the use of hashtags, etc. The sky is really the limit with this project and we are looking to you for thoughts on where to go next.

*NOTE: If you would like to help us mould this tutorial into something amazing, please [email the author](mailto:james+thinkster@jamesbrewer.io) with your thoughts.*

## Who are the Borg?
This is, in our opinion, the most important question of all.

The Borg are an alien race that show up repeatedly in Star Trek, leaving pure distruction in their wake. Their sole goal in life is to assimilate other races into the Borg Collective and to absorb that race's technology in the process. This has led to the Borg being a formidable race with technology well beyond that of the Federation.

While the Borg are conscious of the thoughts of each member of the collective, they are not aware of themselves as individuals. Instead of having names, each Borg is referred to as Third of Five, or similar.

We model this connection by showing every thought of our collective (the set of users in our application). We also provide a way to focus in on the thoughts of a single member of the collective.

## A brief interlude about formatting
Throughout this tutorial, we strive to maintain consistent formatting. This section details what that formatting looks like and what it means.

* Variable names and file names appear in-line with special formatting: `thinkster_django_angular/settings.py`.
* Longer code snippets will appear on their own lines:

        def philz_coffee_is_the_best():
            return True

* Terminal commands also appear on their own line, prefixed by a `$`:

        $ python manage.py runserver

* Unless otherwise specified, you should assume that all terminal commands are run from the root directory of your project.
* Superscript values such as <sup>1</sup> include additional thoughts and information and can be found at the end of each section in **Chapter notes**.

# Setting up your environment
The application we will be building requires a non-trivial amount of boilerplate. Instead of spending time setting up your environment, which is not the purpose of this tutorial, we created a boilerplate project to get you started.

You can find the boilerplate project on Github at [brwr/thinkster-django-angular-boilerplate](https://github.com/brwr/thinkster-django-angular-boilerplate). The repository includes a list of commands you need to run to get everything running.

Go ahead and follow the setup instructions now.

{x: set_up_envrionment}
Follow the instructions to set up your environment

## A humble request for feedback
At the risk of sounding cliche, we would not have a reason to make this tutorial if not for you. Because we believe that your success is our success, we invite us to [contact us](mailto:james@thinkster.io) with any thoughts you have about the tutorial.

We welcome criticism openly and accept praise if you believe it is warranted. We want to know what you like, what you don't like, what you want to know more about, and anything else you feel is relevant.

If you are too busy to reach out to us, that's OK. We know that learning takes a lot of work. If, on the other hand, you want to help us build something amazing, we await your mail.

Let's build some software.

## Chapter notes

*<sup>1</sup> Third of Five is a Star Trek reference to the Borg Hugh before Geordi gave him the name Hugh. You should expect more nerdiness to follow.* 

*<sup>2</sup> Testing takes a considerable amount of time. In true Lean Startup fashion, we have elected not to include testing in the initial release of this tutorial. This was a hard choice, but we want to release quickly and get your feedback. We will be iterating as we get your feedback and the early iterations will include thorough coverage of Django and AngularJS testing.*
