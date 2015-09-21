---
layout: post
title: "React + Flux backed by Rails API - Part 1"
date: 2015-01-28 16:04
comments: true
categories: rails api react flux javascript 
---

I've been working on a frontend for a project we are developing here at Fancy Pixel. We are embracing what looks like a good habit: slicing what would be a monolithic Rails app in a lightweight backend serving APIs and a frontend consuming them. We did this in the not so distant past using Angular.js. It was all fine and dandy, until it wasn't. There's something about it that doesn't sit right with me, I wouldn't go in detail, since many others already did, but let's just say that there's too much magic involved for my tastes (says the guy using Rails). Magic is fine as long as I can figure out how to tinker with the internals when things go down south. With Angular the effort seems too much, but that's just personal taste really. Also I can't deny that the major structural changes introduced in 2.0 were the last nail in the coffin.  
I wanted to try something new, something that would enforce a solid architecture of our apps, letting me control the single cogs in the engine. React got a lot of good press in the past months, so I took the chance to dive in. In this three-part post you'll find pretty much everything I learned by writing a frontend using React, with a vanilla Flux architecture, consuming an API written in Rails. 
<!-- More -->
##Choosing the backend
Given our experience, the obvious choice for us was Rails, but with a twist: [rails-api](https://github.com/rails-api/rails-api). Rails-api is a stripped down version of Rails, where most of the '_useless_' middleware is not included (but you can include it might you need it). Using Rails to serve JSON might seem overkill to most, but the Github page of rails-api has some really good points to counter this argument, and I think they are spot on. 
TL;DR version: Rails is awesome, let's take advantage of that.

##The frontend technology
React is javascript library for building user interfaces built and open sourced by the Facebook's engineers. Its major selling point is the ability to provide a dynamic and fast way to build isomorphic apps.  
Isomorphic means that the app can be rendered with ease on both the server and the client, which helps with SEO.  
Personally, I couldn't care less about SEO, even if I understand how important it is... I was really sold on React by the Virtual DOM and how the data is organized and handled in React views. 
The Virtual DOM is something that we're going to see implemented in other JS frameworks (Ember does that already if I'm not mistaken). The views can be rendered on the server for the initial request, than the underlying tech is going to render subsequent pages in a Virtual DOM, that is then diffed with the actual DOM, and then only the differences are changes in the visible page. And it's fast. Brilliant. This enables us to start writing frontend like we used way back: in a declarative way... we just specify how the UI should look, when data changes React takes care of the page refresh, changing only the parts that need to be changed. 

##Flux
This covers the backend and the views, we're missing something in between, say, an architecture to follow. Flux is an architecture for building web UIs, and works really well in combination with React (but it can really be applied anywhere). 

##Here comes trouble
I never was a big fan of implementing web UI, CSS always gets messy, Javascript files become scary monoliths where crappy code goes to die, while developers test their spelunker skills and loose their sanity. Maybe I'm just crap, but even using Sass and Coffeescript never really solved my issues.  
I was excited to try something really new, but I knew that getting started with such a young technology would end up being a major pain in the ass.   
Case and point, learning and starting being productive (i.e.: writing usable code) took a fair bit of head scratching. There's still no clear "best practice" to perform common tasks, nor a clear starting configuration. Let's put it that way, if you come from the RoR world, where convention over configuration greatly reduces boilerplating and "forces" you to follow commonly established best practice, you're going to struggle with Flux. 
This post will cover the solutions we came up with, they may not be perfect, but I'm fairly sure there are no anti-patterns in there, and they are a solid starting point.

#Getting it all toghether
Let's start writing some code. We'll go through a simple Rails app that will feature user signup and login, and the ability to post a story. Just like a tiny Medium. Let's appropriately call it _Small_.  

Feel free to skip the Railsy part if you're only interested in Flux and React and jump to [Part 2](/blog/2015/01/29/react-plus-flux-backed-by-rails-api-part-2/).

#Rails API
A while ago I stumbled upon [this article](http://slides.com/alanpeabody/breaking-up-with-the-asset-pipeline#/) by Alan Peabody. I had a similar experience as him, you start working ona project, you use the right tools, you do your best to enforce good patterns, but in the end the frontend code just becomes scary, something no one wants to maintain. Let's break up with the asset pipeline, as the title says, and work towards making Rails beautiful again. 
We'll be using the rails-api gem for this task. You can generate a new app with its CLI command, or you can integrate it later. I'll do the later option, no reason really, just a habit.

~~~
rails new small
~~~

Next we'll add `rails-api`, `devise`, `active_model_serializers` gems to our Gemfile, and while we are at it we can remove all the gems that generate assets or view content, `jbuilder` included. Our Gemfile should look like this (test section omitted):

~~~ruby
source 'https://rubygems.org'

gem 'rails', '4.2.0'
gem 'rails-api', '~> 0.4.0'
gem 'active_model_serializers', '~> 0.8.3' # NOTE: not the 0.9
gem 'devise', '~> 3.4.1'
gem 'sqlite3'
gem 'sdoc', '~> 0.4.0', group: :doc
gem 'thin'

group :development, :test do
  gem 'faker'
  gem 'byebug'
  gem 'web-console', '~> 2.0'
  gem 'spring'
end
~~~

Now we need to change the application controller so that it inherits from `ActionController::API`, and kiss the `protect_from_forgery` goodbye. Since we are serving only JSON, it makes sense to add 

~~~ruby
respond_to :json
~~~

to the applciation controller, helps DRYing all out. While we are at it, we might as well delete the assets and views folders, we won't need them.

##Authentication
Should I first define a resource? Maybe, but that's trivial, let's get the authentication out of the way. We are building an API, so no session will be involved, we have to authenticate the user in each request. I'll be using [Oauth2 Resource Owner Password Credentials Grant](http://oauthlib.readthedocs.org/en/latest/oauth2/grants/password.html) which sounds fancy, but it's really just a token in the request header that authenticates the caller.  
The gem Devise used to implement a `token_authenticatable` strategy, but it was pulled for security reason. There are gems that implement the strategy (like Doorkeeper), but since it's fairly easy to implement I'll do it for myself. Let's install Devise first by adding it in the Gemfile and launching `rails generate devise:install` after a `bundle install`, then we create the user model:

~~~
rails generate devise User
~~~

##Token authentication
Token authentication was removed from Devise a couple of years ago, [this link](http://blog.plataformatec.com.br/2013/08/devise-3-1-now-with-more-secure-defaults/) explains why. We have to implement it for ourselves, but it's quite easy. The token will be composed of two informations: the user's id followed by the token itself, separated by a `:`. We'll be using the user's database id for this sample, for semplicity's sake, but it's obviously not a smart thing to do. 
First things first, we'll add an `access_token` to the user (and a username too):

~~~ruby
class AddAccessTokenToUser < ActiveRecord::Migration
  def change
    add_column :users, :access_token, :string
    add_column :users, :username, :string
  end
end
~~~

and here's the User model:

~~~ruby
# app/models/user.rb
class User < ActiveRecord::Base
  devise :database_authenticatable, :recoverable, :validatable

  after_create :update_access_token!  

  validates :username, presence: true
  validates :email, presence: true

  private

  def update_access_token!
    self.access_token = "#{self.id}:#{Devise.friendly_token}"
    save
  end
  
end
~~~

The user authentication will sit in the application controller:

~~~ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include AbstractController::Translation

  before_action :authenticate_user_from_token!
  
  respond_to :json

  ## 
  # User Authentication
  # Authenticates the user with OAuth2 Resource Owner Password Credentials Grant
  def authenticate_user_from_token!
    auth_token = request.headers['Authorization']

    if auth_token
      authenticate_with_auth_token auth_token
    else
      authentication_error
    end
  end

  private

  def authenticate_with_auth_token auth_token 
    unless auth_token.include?(':')
      authentication_error
      return
    end

    user_id = auth_token.split(':').first
    user = User.where(id: user_id).first

    if user && Devise.secure_compare(user.access_token, auth_token)
      # User can access
      sign_in user, store: false
    else
      authentication_error
    end
  end

  ## 
  # Authentication Failure
  # Renders a 401 error
  def authentication_error
    # User's token is either invalid or not in the right format
    render json: {error: t('unauthorized')}, status: 401  # Authentication timeout
  end
end
~~~

We conclude the auth process by providing the routes and the session controller:

~~~ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :user, only: []
  
  namespace :v1, defaults: { format: :json } do
    resource :login, only: [:create], controller: :sessions
  end
end
~~~

and the session controller:

~~~ruby
# app/controllers/v1/sessions_controller.rb
module V1
  class SessionsController < ApplicationController
    skip_before_action :authenticate_user_from_token!
    
    # POST /v1/login
    def create
      @user = User.find_for_database_authentication(email: params[:username])
      return invalid_login_attempt unless @user

      if @user.valid_password?(params[:password])
        sign_in :user, @user
        render json: @user, serializer: SessionSerializer, root: nil
      else
        invalid_login_attempt
      end
    end

    private

    def invalid_login_attempt
      warden.custom_failure!
      render json: {error: t('sessions_controller.invalid_login_attempt')}, status: :unprocessable_entity
    end

  end
end
~~~

The SessionSerializer is an Active Model Serializer object, something like this:

~~~ruby
# app/serializers/v1/session_serializer.rb
module V1
  class SessionSerializer < ActiveModel::Serializer

    attributes :email, :token_type, :user_id, :access_token

    def user_id
      object.id
    end

    def token_type
      'Bearer'
    end

  end
end
~~~

That's it. Migrate, run the server, and create a user via the console. You should get something like this:

~~~
$ curl localhost:3000/v1/login --ipv4 --data "usernme=user@example.com&password=password"
{
  "token_type": "Bearer",
  "user_id": 1,
  "access_token": "1:MPSMSopcQQWr-LnVUySs"
}
~~~

##Creating a resource
I won't go in detail here, the task is just plain RoR. We'll create a Story resource and a controller that will handle the user creation. You'll find the complete rails app in [this repo](https://github.com/FancyPixel/small-rails). Moving on.

##CORS
It's worth noting that the UI now will not be served by rails, it might even sit in a different server. There are solution that let us keep both UI and API on the same domain, but for now we will need to enable Cross-origin resource sharing ([CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing)). We can do this by adding the [rack-cors gem](https://github.com/cyu/rack-cors) to our Gemfile and then add this in our `config/application.rb`:

~~~ruby
config.middleware.insert_before 'Rack::Runtime', 'Rack::Cors' do
  allow do
    origins '*'
    resource '*',
             headers: :any,
             methods: [:get, :put, :post, :patch, :delete, :options]
  end
end
~~~
This opens up to every domain, so thread lightly.

##Next up
That's it for the Rails part. It's such a joy writing just the API in Rails. It's easier to test, easier to maintain and it's blazing fast. Now we'll proceed with the frontend. 
For readability's sake I'll split the article here, jump to Part 2 to start building the frontend. 

[Part 2](/blog/2015/01/29/react-plus-flux-backed-by-rails-api-part-2/)

