---
layout: post
title: "APIs with Rails:<br/>render :json => the_simple_way"
date: 2015-05-23 14:50:41 +0200
comments: true
categories: ruby rails api json
author: Alessandro Verlato (@Aleverla)
published: true
---

Lately my work at [Fancy Pixel](http://fancypixel.it) has focused on the backend of a product we're about to launch (link Space Bunny?) and for which we decided to build a JSON API-only server. These APIs can be consumed from third party clients/services but are also used by our frontend. In this short report I'd like to share with you the simple solution that we're using for the JSON generation and that in my humble opinion can be an easy trick alternative to most used systems like e.g. Jbuilder and ActiveModel::Serializers.

<!-- More -->

Rails, one of my favourite work buddies with which it happened several times to work on projects which included the development of APIs. Due to personal curiosity, during the years I had the opportunity to try several solutions for JSON generation and I have to say that sometimes I found some difficulties with certain tecnologies: wonderful for the majority of their functionalities sometimes they may force you to take weird paths and make forcing to achieve the desired result. To be honest, the main reason for this continuative experimentation is probably the constant search for the top balancing between comfort/easiness/development-speed and performances and after all this test and try I arrived to the current solution, that probably is not nothing new, but that in my opinion can maybe give somebody an alternative idea combining together great flexibility and bare metal performances.

## To cut a long story short... 

I don't want to dwell with epic tales or "disquisitions about the gender of the Angels" so I built a trivial [demo app](https://github.com/FancyPixel/serializers_demo) that you can find on our GitHub account, with which show you what I'm talking about.

Let's rapidly setup our application:

~~~bash
git clone "https://github.com/FancyPixel/serializers_demo"
cd serializers_demo
bundle
rake db:create && rake db:migrate && rake db:seed
~~~

For the purposes of this article I decided to use [rails-api](https://github.com/rails-api/rails-api) (if you don't already know it I reccomend you to give it a try) instead of standard Rails, for the simple fact that this is what we're using right now in the project I talked to you at the beginning of this article. Obviously the same concepts apply identical to "pure" Rails.
Let's open together the code and let's take a rapid look on it: as you can see these few lines of source do nothing but respond to three routes and if you throw a look at ```config/routes.rb``` you'll find something like:

~~~ruby
# config/routes.rb

Rails.application.routes.draw do
  namespace :v1, defaults: { format: :json } do
    get :jbuilder, to: 'comparison#jbuilder'
    get :ams, to: 'comparison#ams'
    get :simple, to: 'comparison#simple'
  end
end 
~~~

As you can see I've set ```json``` as default format and I defined a namespace in such a way to replicate a typical process of APIs versioning.
Let's jump to the only existing controller (```comparison_controller```) where we find the implementation of the actions called from the routes. Each of these actions does exactly the same: load a bunch of records from the DB and render it as JSON, but each one does the rendering in his own way i.e. using respectively jbuilder, ActiveModel::Serializers and "my solution" that I'm going to call "simple"...  what a fancy name uh?

We're not going to focus on the first two systems, because chances are that you master those tecnologies better than me and for more there isn't nothing out of stantard in my implementations, but instead we're jumping feet together to the ```simple``` action. As for competitors before, she doesn't nothing but render some JSON, but this calling ```serialize_awesome_stuffs``` in help. This method is defined in the ```V1::SimpleAwesomeStuffSerializer``` module that is included by the controller. You can find the module under ```app/serializers/v1``` and if you're going to open the file you'll notice that it's nothing but a normal Ruby module defining methods.

~~~ruby
# app/serializers/v1/simple_awesome_stuff_serializer.rb

module V1
  module SimpleAwesomeStuffSerializer

    def serialize_awesome_stuff(awesome_stuff = @awesome_stuff)
      {
          name: awesome_stuff.name,
          some_attribute: awesome_stuff.some_attribute,
          a_counter: awesome_stuff.a_counter
      }
    end

    def serialize_awesome_stuffs(awesome_stuffs = @awesome_stuffs)
      {
          awesome_stuffs: awesome_stuffs.map do |awesome|
            serialize_awesome_stuff awesome
          end
      }
    end
  end
end
~~~

Both the methods do nothing more that returning an Hash defining key-values couples that we're willing to return as JSON. In particular ```serialize_awesome_stuffs``` creates an ```Array``` of ```Hash``` and internally, just for DRYing things up a bit, it calls ```serialize_awesome_stuff``` (singular). Maybe the overall naming is not the best in the world, uh? Bonus point: defining the method's parameter ```awesome_stuffs = @awesome_stuffs``` allows us to make lighter and more readable our code because if we remained adherent to conventional variable naming, probably our controller is defining something like ```@awesome_stuff``` (and you saw that we did) that is directly visible and usable by our module. If we're going to have a bout of creativity and we'll want to use our personal variables names, we'll not have any sort of problem.

Take this piece of code as example:

~~~ruby
# some_controller.rb

def some_action
  @my_personal_awesome_stuffs = AwesomeStuff.all
  render json: serialize_awesome_stuffs @my_personal_awesome_stuffs
end
~~~

and everything will work as expected.

## Step-by-step: let's complicate things a bit 

Let's go together raise a bit the complexity of our example, adding a ```User``` model and defining a one-to-many relationship with our AwesomeStuff:

~~~bash
rails g model user name:string
~~~

add the User reference in AwesomeStuff:

~~~bash
rails g migration add_user_reference_to_awesome_stuff user:references
~~~

and migrate everything:

~~~bash
rake db:migrate
~~~

Define now the relationships between models:

~~~ruby
# app/models/awesome_stuff.rb
class AwesomeStuff < ActiveRecord::Base
  belongs_to :user
end

# app/models/user.rb
class User < ActiveRecord::Base
  has_many :awesome_stuffs
end
~~~

launch the Rails console with ```rails c``` and insert some test data into the DB:

~~~ruby
# Add five users
users = []
5.times {|n| users << User.create(name: "user_#{n}") }

# Randomly associate our awesome records™ to users
AwesomeStuff.all.each { |aww| aww.update user: users.sample }

# A rapid test confirms that...
User.first.awesome_stuffs

=> #<ActiveRecord::Associations::CollectionProxy [#<AwesomeStuff id: ... ... >]>
~~~

Now that everything is prepared, let's follow some of the steps we'd usually do during an API creation. Let's create a ```UserController``` through which return to the client also user's associated awesome records™.
Create ```users_controller.rb` under ```app/controllers/v1/``` and add ```index``` action:

~~~ruby
# app/controllers/v1/users_controller.rb

module V1
  class UsersController < ApplicationController
    include V1::UsersSerializer

    def index
      @users = User.all.includes(:awesome_stuffs)
      render json: serialize_users
    end
  end
end
~~~

As you can see I already added some stuff that we'll need in short, that is ```V1::UsersSerializer``` module. If you haven't already, notice the scoping (V1) of our serializers: in doing so we can follow the evolution of our API's versions with no hassles, possibly going to redefine the behavior of only serializers that may change.
Do not forget to add new routes:

~~~ruby
# config/routes.rb

Rails.application.routes.draw do
  namespace :v1, defaults: { format: :json } do
    get :jbuilder, to: 'comparison#jbuilder'
    get :ams, to: 'comparison#ams'
    get :simple, to: 'comparison#simple'

    # Let's add the 'index' only
    resources :users, only: :index
  end
end
~~~

What are we going to add to our ```UsersSerializer```? A first idea should be something like:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          {
              id: user.id,
              name: user.name,
              awesome_stuffs: user.awesome_stuffs.map do |aww|
                {
                    name: aww.name,
                    some_attribute: aww.some_attribute,
                    a_counter: aww.a_counter
                }
              end
          }
        end
      }
    end
  end
end
~~~

Ok, but there's a lot of code smell here right? We already saw some of this stuff, let's try to reuse it:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          {
              id: user.id,
              name: user.name,
              awesome_stuffs: user.awesome_stuffs.map do |aww|
                serialize_awesome_stuff aww
              end
          }
        end
      }
    end
  end
end
~~~

Very well, but we can do better. Our API will probably have a route for a user's data i.e. something like ```/v1/users/1``` so we can move in advance and simultaneously dry up our current code:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_user(user = @user)
      {
          id: user.id,
          name: user.name,
      }
    end

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          serialize_user(user).merge(
            {
                awesome_stuffs: user.awesome_stuffs.map do |aww|
                  serialize_awesome_stuff aww
                end
            }
          )
        end
      }
    end
  end
end
~~~

Ok, we've just "killed two birds with one stone". (the birds are doing well, don't worry)
As you'll have noticed it's possible to obtain a further improvement:

~~~ruby
# app/serializers/v1/users_serializer.rb

module V1
  module UsersSerializer
    include V1::SimpleAwesomeStuffSerializer

    def serialize_user(user = @user)
      {
          id: user.id,
          name: user.name
      }
    end

    def serialize_users(users = @users)
      {
        users: users.map do |user|
          serialize_user(user).merge(serialize_awesome_stuffs(user.awesome_stuffs))
        end
      }
    end
  end
end
~~~

What you've seen so far is a simple example of what it's possibile to do with the tools we already have at hand and mainly wants to be an idea for those wo are constantly looking for the best performances and simplicity.

We got to the end and I hope to not have bored you, but if you come to read up here maybe so :)
What you have seen today may or may not be liked, but I personally find it a surely performant system that offers pure modularity, extensibility and code reuse.

Feel free to leave a comment, we’d really love to hear your feedback.

See ya soon!

Alessandro - @madAle
