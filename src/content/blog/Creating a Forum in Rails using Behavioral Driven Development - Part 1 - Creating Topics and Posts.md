---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 1 - Creating Topics and Posts"
description: "Creating topics and posts for a forum in Rails"
pubDate: "Mar 25 2017"
---

I'll be creating a forum from scratch for a client in this blog post.  In the application I have already created users with devise so I won't be going through that process.

## Table of Contents

- [Creating Topics and Posts](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-1---creating-topics-and-posts/)
- [Creating Comments](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-2---creating-comments/)
- [Editing the Models](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-3---editing-the-models/)
- [Deleting the Models](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-4---deleting-the-models/)
- [Styling Topics](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-5---styling-and-adding-information-to-topics/)
- [Styling Posts Listing](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-6---styling-and-adding-information-to-posts-listing-topic-page/)
- [Adding a post Partial](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-7---adding-a-post-partial/)
- [Styling Comments](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-8---styling-and-adding-information-to-comments-listing-post-page/)

## Models

### Laying out the relationships
- `Users` has_many `Posts` and `Comments`
- `Posts` belongs_to `Users` and `Topics` and has_many `Comments`
- `Comments` belongs_to `Posts` and `Users` and belongs_to and has_many `Comments` of their own
- `Topics` has_many `Posts`

### Creating the Post, Topic and Comment models
```irb
rails g migration create_posts title:string content:text user:references topic:references
rake db:migrate
rails g migration create_topics name:string
rake db:migrate
rails g migration create_comments comment:text post:references user:references comment:references
rake db:migrate
```

### Factories
I use FactoryGirl and Faker to create the models for my tests.  I have already created a factory for users but will create one for each other model in this tutorial.  When creating a factory you need to create an association for every model that the factory belongs to.  In the case of Post it will need an association for Topic and User models.

*spec/factories/post.rb*
```ruby
require 'faker'

FactoryGirl.define do
  factory :post do
    association :user, factory: :user
    association :topic, factory: :topic
    title { Faker::Lorem.sentence }
    content { Faker::Lorem.paragraph }
  end
end
```

I also need to create the Topic factory for the above to be valid.

*spec/factories/topic.rb*
```ruby
require 'faker'

FactoryGirl.define do
  factory :topic do
    name {Faker::Lorem.word }
  end
end
```

### Model Files
Creating the model for Topic with the association for posts.  The name must also be unique as there is no point in multiple topics with the same name.

*app/models/topic.rb*
```ruby
class Topic < ApplicationRecord
  has_many :posts
  validates :name, presence: true
  validates_uniqueness_of :name
end
```

Creating the model for Post I'll make the associations I mentioned above.  We don't want to destroy the comments of a Post if it is deleted since it will potentially destroy other users content if they put a lot of work into their comment.  The comments will just live unlisted within the users profile. I will also set character limits for both the title and the content of the post.

*app/models/post.rb*
```ruby
class Post < ApplicationRecord
  belongs_to :user
  belongs_to :topic
  has_many :comments

  validates :title, presence: true, length: {minimum: 2, maximum: 200}
  validates :content, presense: true, length: {minimum: 2, maximum: 40000}
  validates :user_id, presence: true
  validates :topic_id, presence: true
end
```

### Model Specs
We want to test if these factories are valid now.  We will start with the Topic factory since it is required for the Post factory.

*spec/models/topic_spec.rb*
```ruby
require 'rails_helper'

RSpec.describe Topic, type: :model do
  it "has a valid factory" do
    expect(FactoryGirl.build(:topic)).to be_valid
  end
end
```

*spec/models/post_spec.rb*
```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  it "has a valid factory" do
    expect(FactoryGirl.build(:post)).to be_valid
  end
end
```

Now we can run rspec to make sure we have valid factories for posts and topics.
```irb
rspec
```

This gave me an error which I run into on a fairly consistent basis.
```irb
SQLite3::BusyException: database is locked:
```

From [this Stackoverflow answer](http://stackoverflow.com/questions/7154664/ruby-sqlite3busyexception-database-is-locked) I run this command and it fixes it right up
```irb
rails console
ActiveRecord::Base.connection.execute("BEGIN TRANSACTION; END;")
exit
```

Now we have 0 errors, and we are ready to dive into feature tests.

## Feature Specs
### Creating Topics
Pseudo code for creating a Topic
- Administrator logs in and clicks on the forum link in the navigation bar
- Administrator clicks "Create Topic"
- Administrator fills in a name for a topic and hits submit.

When creating the feature test I wanted to log in as an administrator within a block then do all tests nested within, then do the same thing as a regular user so I could keep the code DRY.

Looking at [The capybara documentation](https://github.com/teamcapybara/capybara) it didn't give any nested examples in the README.  So I found [this stackoverflow answer](http://stackoverflow.com/questions/15253091/nested-feature-in-capybara-2-0) where it states the regular nested test goes like the following:

```ruby
describe "something" do
  context "under a certain condition" do
    it "does something" do
    end
  end
end
```

The Capybara documentation stated that `feature` is an alias for `describe ..., :type => :feature`, and `scenario` is an alias for `it`.  So there is no alias for `context`, but between the above stackoverflow answer and [this example](https://tirdadc.github.io/blog/2014/09/11/nested-feature-tests-with-capybara-and-rspec/) it seems the best practice is to nest `feature` within `feature` until you end with a `scenario`.  Logging in, clicking buttons and links and probably a few other Capybara methods also can't reside within a `feature` block on their own, and need to always be within a `before` block.  I finally end up with the following test that is giving us real errors and not just syntax errors.

*spec/features/topic/create_topic_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Creating a Topic" do
  before do
    @user = FactoryGirl.create(:user)
    @admin = FactoryGirl.create(:admin)
    @topic = FactoryGirl.build(:topic)
  end

  feature "as an administrator" do
    before do
      login_as(@admin)
      visit "/"
      click_link "Forum"
      click_button "New Topic"
    end

    scenario "with valid inputs" do
      fill_in "Name", with: @topic.name
      click_button "Create Topic"
      expect(page).to have_content("A new topic has been successfully created")
      expect(page).to have_content(@topic.name)
    end

    scenario "with invalid inputs" do
      fill_in "Name", with: ""
      click_button "Create Topic"
      expect(page).to have_content("Name can't be blank")
    end
  end

  feature "as a regular user" do
    login_as(@user)

    scenario "through the user interface" do
      visit "/"
      click_link "Forum"
      expect(page).not_to have_content("New Topic")
    end

    scenario "by going directly to the route" do
      visit "/topics/new"
      expect(current_path).to eq(root_path)
    end
  end
end
```

We get the following locations of the errors which read very nicely.
```irb
rspec ./spec/features/topics/create_topic_spec.rb:18 # Creating a Topic as an administrator with valid inputs
rspec ./spec/features/topics/create_topic_spec.rb:25 # Creating a Topic as an administrator with invalid inputs
rspec ./spec/features/topics/create_topic_spec.rb:34 # Creating a Topic as a regular user through the user interface
rspec ./spec/features/topics/create_topic_spec.rb:40 # Creating a Topic as a regular user by going directly to the route
```

The first real error we can start working with is.
```irb
No route matches [GET] "/topics/new"
```

In the interest of saving time so we don't add all of the routes individually we can add everything that topics needs right now.  We don't want the ability to delete a topic since it is the only place that posts reside and it would require deleting each post underneath the topic.  We also don't want to be able to edit a topic, since changing it would make all the posts within not relevant.  In an emergency we can always implement these actions from the command line.  That just leaves us with create, new, and index.

*config/routes.rb*
```ruby
resources :topics, only: [:create, :new, :index]
```

Next error:
```irb
uninitialized constant TopicsController
```

Again, we can make the controller functional enough to already implement the show, new, create, and index actions.

*app/controllers/topics_controller.rb*
```ruby
class TopicsController < ApplicationController
  include ApplicationHelper
  before_action :require_admin, except: [:show, :index]

  def show
    @topic = Topic.find(params[:id])
  end

  def index
    @topics = Topic.all
  end

  def new
    @topic = Topic.new
  end

  def create
    @topic = Topic.new(topic_params)
    if @topic.save
      flash[:success] = "A new topic has been successfully created"
      redirect_to topics_path
    else
      render 'new'
    end
  end

  private

  def topic_params
    params.require(:topic).permit(:name)
  end
end
```

Next error:
```irb
TopicsController#new is missing a template for this request format and variant.
```

*app/views/topics/new.html.erb*
```erb
<%= render 'shared/errors', obj: @topic %>
<%= form_for(@topic) do |f| %>
  <div class="form-group">
    <%= f.label :name %>
    <%= f.text_field :name, class: "form-control" %>
  </div>
  <div class="form-group">
    <%= f.submit class: "btn btn-primary" %>
  </div>
<% end %>
```

Next error:
```irb
Failure/Error: expect(current_path).to eq(root_path)
       expected: "/"
            got: "/topics/new"
```

We haven't implemented the require_admin function yet inside the controller.  I suspect we will need this for other controller so i put it inside of application_helper.

*app/helpers/application_helper.rb*
```ruby
def require_admin
  current_user.nil? ? redirect_to(root_path) : (redirect_to(root_path) unless current_user.admin?)
end
```

*app/controllers/topics_controller.rb*
```ruby
include ApplicationHelper
before_action :require_admin, except: [:show, :index]
```

Next error:
```irb
Unable to find link "Forum"
```

Our forum link is just going to be the index action of our topics.

*app/views/layouts/_navbar.html.erb*
```erb
    <li>
      <%= link_to "Forum", topics_path %>
    </li>
```

We can go ahead and make the template since the next error will be template is missing.

*app/views/topics/index.html.erb*
```erb
  <%= link_to "New Topic", new_topic_path, class: "btn btn-primary" %>
```

Next error:
```irb
Creating a Topic as a regular user through the user interface
     Failure/Error: expect(page).not_to have_content("New Topic")
```

We need to wrap the above button with a clause that it only appears for administrators.

*app/views/topics/index.html.erb*
```erb
<% if current_user.try(:admin?) %>
  <%= link_to "New Topic", new_topic_path, class: "btn btn-primary" %>
<% end %>
```

Next error:
```irb
Unable to find button "New Topic"
```

I forgot that links with a button class are still links, so in the spec I changed the "New Topic" instance of click_button to click_link.

Next error:
```irb
Creating a Topic as an administrator with valid inputs
     Failure/Error: expect(page).to have_content(@topic.name)
```

Just to get the test passing and will focus on styling the index page later

*app/views/topics/index.html.erb*
```erb
<% @topics.each do |topic| %>
    <%= topic.name %>
    <br>
<% end %>
```

That's everything passing for Topics and now we can move on to Posts.

### Creating Posts

Pseudo code for creating a Post
- User logs in or creates an account then clicks on the forum link in the navigation bar
- User clicks on one of the topics such as general or introduction
- User sees a list of all posts under that topic, then scrolls down and clicks to create a new post
- User fills in the text area to create their post, then clicks submit
- User is redirected to their post where they have the option of editing or deleting their post

The skeleton for this spec will be the same as creating a topic spec.  There will be little variance in the model that is used when testing its creation.

*spec/features/posts/create_post_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Creating a Post" do
  before do
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.build(:post)
  end

  feature "as a user" do
    before do
      login_as(@user)
      visit "/"
      click_link "Forum"
      click_link "#{@topic.name}"
      click_link "New Post"
    end

    scenario "with valid inputs" do
      fill_in "Title", with: @post.title
      fill_in "Content", with: @post.content
      click_button "Create Post"
      expect(page).to have_content("Your post has been successfully created")
      expect(page).to have_content(@post.title)
      expect(page).to have_content(@post.content)
    end

    scenario "with invalid inputs" do
      fill_in "Title", with: ""
      fill_in "Content", with: ""
      click_button "Create Post"
      expect(page).to have_content("Title can't be blank")
      expect(page).to have_content("Content can't be blank")
    end
  end

  feature "as a guest" do

    scenario "through the user interface" do
      visit "/"
      click_link "Forum"
      click_link "#{@topic.name}"
      expect(page).to have_content("You must sign in or sign up to view this page")
      expect(current_path).to eq(new_user_registration_path)
    end

    scenario "by going directly to the route" do
      visit "/topics/#{@topic.id}/posts/new"
      expect(page).to have_content("You must sign in or sign up to view this page")
      expect(current_path).to eq(new_user_registration_path)
    end
  end
end
```

First error
```irb
     Failure/Error: click_link "#{@topic.name}"

     Capybara::ElementNotFound:
       Unable to find link "velit"
```

Velit is the name that Faker generated for the topic name.  We need to turn the name in the topic index into a link which leads to the topic itself, which in turn will hold all of the posts which belong to it.

*app/views/topics/index.html.erb*
    <%= link_to "#{topic.name}", topic_path(topic) %>
```

This gives us the error
```irb
     Failure/Error: <%= link_to "#{topic.name}", topic_path(topic) %>

     ActionView::Template::Error:
       undefined method `topic_path' for #<#<Class:0x000000082d4170>:0x0000000812cfc0>
       Did you mean?  topics_path
```

We didn't create the route for showing a topic when we were just thinking about creating the topic itself.  Even though we made the action in the controller.  It's a quick fix to add this method into the topic routes:.  I will also create the associated file *app/views/topics/show.html.erb* so we can have a more meaningful error afterwards.

*config/routes.rb*
```ruby
resources :topics, only: [:create, :new, :index, :show]
```

After creating the show file we get the following error

```irb
Unable to find link "New Post"
```

*app/views/topics/show.html.erb
```erb
<%= link_to "New Post", new_post_path, class: "btn btn-primary" %>
```

which gives us the following:

```irb
undefined local variable or method `new_post_path'
```

Posts have to be able to undergo all CRUD actions.  However we never want to index all posts as they will all be under different topics.  We also want the post route to be nested within topics so we need to embed it within a do block.

*config/routes.rb*
```ruby
  resources :topics, only: [:create, :new, :index, :show] do
    resources :posts, except: [:index]
  end
```

We are sitll getting the same error.  Which is because we forgot to preemtively change the path to take into account the nested route.  We need to change it to new_topic_post_path(topic) and embed it within the @topics loop.

*app/views/topics/index.html.erb*
```erb
<% @topics.each do |topic| %>
    <%= link_to "#{topic.name}", topic_path(topic) %>
    <br>
  <% if current_user.try(:admin?) %>
    <%= link_to "New Topic", new_topic_post_path(topic), class: "btn btn-primary" %>
  <% end %>
<% end %>
```





*app/views/topics/show.html.erb
```erb
<%= link_to "New Post", new_topic_post_path, class: "btn btn-primary" %>
```

We can go ahead and make the controller since we need the associated methods for creating a post.

*app/controllers/posts_controller.rb*
```ruby
class PostsController < ApplicationController

  def new
    @post = Post.new
  end

  def create
    @post = Post.new(post_params)
    if @post.save
      flash[:success] = "Your post has been successfully created"
      redirect_to post_path(@post)
    else
      render 'new'
    end
  end

  private
  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

We can also make the new template in order to make the next error more meaningful.  This gives us the following

```irb
     Failure/Error: fill_in "Title", with: @post.title

     Capybara::ElementNotFound:
       Unable to find field "Title"
```

Since we want to be able to create and edit posts, we will put the form itself into a form partial.  We need both @topic and @post in the form_for since they are nested

*app/views/posts/_form.html.erb*
```erb
<%= render 'shared/errors', obj: @post %>
<%= form_for([@topic, @post]) do |f| %>
  <div class="form-group">
    <%= f.label :title %>
    <%= f.text_field :title, class: "form-control" %>
  </div>
  <div class="form-group">
    <%= f.label :content %>
    <%= f.text_area :content, class: "form-control" %>
  </div>
  <div class="form-group">
    <%= f.submit class: "btn btn-primary" %>
  </div>
<% end %>
```

*app/views.posts/new.html.erb*
```erb
<h2>Create a new Post</h2>
<%= render 'form' %>
```

This gives the following error
```irb
     Failure/Error: <%= form_for([@topic, @post]) do |f| %>

     ActionView::Template::Error:
       undefined method `posts_path' for #<#<Class:0x00000007a6bd58>:0x000000096949a8>
       Did you mean?  topics_path
```

I tried to figure out exactly why this form was trying to use the index path which doesn't exists for the new post form and couldn't figure it out.  So I just set the url to be the create method path topic_posts_path.

*app/views/posts/_form.html.erb*
```erb
<%= form_for([@topic, @post], url: topic_posts_path) do |f| %>
```

We now get an error saying it couldn't find the success message.  The relevant errors were:

```irb
User must exist
Topic must exist
User can't be blank
Topic can't be blank
```

We can set the user and the topic in the create action of the controller.  The user is simply the current_user and the topic we can find in the url using params.  We can also change the post_path to topic_post_path in advance for our updates routes.

*app/controllers/posts_controller.rb*
```ruby
  def create
    @post = Post.new(post_params)
    @topic = Topic.find(params[:topic_id])
    @post.user = current_user
    @post.topic = @topic
    if @post.save
      flash[:success] = "Your post has been successfully created"
      redirect_to topic_post_path(@topic, @post)
    else
      render 'new'
    end
  end
```

Moving on to the next error
```irb
     Failure/Error: click_button "Create Post"

     AbstractController::ActionNotFound:
       The action 'show' could not be found for PostsController
```

We can go ahead and make the action and an empty show.html.erb file.

*app/controllers/posts_controller.rb*
```ruby
  def show
    @post = Post.find(params[:id])
  end
```

We then get the error that it can't find the faker generated title and content
```irb
expected to find text "Earum et ad ducimus et consequatur ab eligendi in."
```

Again we will add more content and style these pages later, but for now I am just working on getting the tests passing

*app/views/posts/show.html.erb*
```ruby
<h2><%= @post.title %></h2>
<%= @post.content %>
```

Next error:
```irb
Failure/Error: expect(page).to have_content("You must sign in or sign up to create a post")
       expected to find text "You must sign in or sign up to create a post"
```

I used the devise method authenticate_user! for other controllers in this application howevever there is no associated error message and I am not able to customize the redirect

I found [this link](https://solidfoundationwebdev.com/blog/posts/redirect-to-specific-page-if-user-is-not-logged-in) which explains how to customize this method.

*app/controllers/application_controller.rb*
```ruby
  private
  def authenticate_user!
    if user_signed_in?
      super
    else
      redirect_to new_user_registration_path, alert: "You must sign in or sign up to view this page"
    end
  end
```

We then have to implement the method into our controller

*app/controllers/posts_controller.rb*
```ruby
  before_action :authenticate_user!, except: [:show]
```

Finally all tests pass and we can move onto comments [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-2---creating-comments/)

