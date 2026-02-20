---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 2 - Creating Comments"
description: "Creating comments for a forum in Rails"
pubDate: "Apr 18 2017"
---

Following the same format as part 1, we will start off with creating the factory, model file, and model spec.  Since we will be dealing with 2 different types of comments: The Comment replying to a post and the comment replying to a comment, we should have a factory for each so we can properly test each of them.  The child comment will inherit from the comment factory with the addition of an association between the two.

Before we kick things off, I realize I set the attribute name for the comment to also be comment, which while might be fine, doesn't really make much sense for differentiating between attributes and will be a pain to debug.  So I am going to change it to content to match the post attribute.  The other thing we need to change is have the id relating the comments be parent_id instead of comment_id.  Luckily the last migration we did was the comments table so we can simply roll it back and create a new one.  This was found through [this stackoverflow post](http://stackoverflow.com/questions/19711033/how-to-write-a-migration-for-a-rails-model-that-belongs-to-itself).  Make sure to delete the old migration file.

```irb
rails db:rollback
rails generate migration create_comments
```

change the contents of the migration file to the following:

```ruby
class CreateComments < ActiveRecord::Migration[5.0]
  def change
    create_table :comments do |t|
      t.text :content
      t.references :post, foreign_key: true
      t.references :user, foreign_key: true
      t.references :parent_comment, foreign_key: true
    end
  end
end
```

```irb
rails db:migrate
```

Now we can move on to the factory.  We need both a comment factory and a child_comment factory since we need to test the creation of those 2 models seperately.  The child comment will be created differently and be tested for a parent_comment id.

## Model

### Factory

*spec/factories/comments.rb*
```ruby
require 'faker'

FactoryGirl.define do
  factory :comment do
    association :user, factory: :user
    association :post, factory: :post
    content { Faker::Lorem.paragraph }
  end

  factory :child_comment, parent: :comment do
    association :parent_comment, factory: :comment
  end
end
```

### Model File

For the model file, we will be requiring everything except for the comment_id since the comment can live underneath a post with no parent comment.  I had to look up how to have a model require or have many of itself within the model file and found [this stackoverflow post](http://stackoverflow.com/questions/18791874/rails-model-has-many-of-itself).  It explains how you set up it to require or have many of a different object with the same class while refering to the foreign key.

*app/models/comment.rb*
```ruby
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :post
  belongs_to :parent_comment, :class_name => "Comment", :foreign_key => "parent_comment_id"
  has_many :child_comments, :class_name => "Comment", :foreign_key => "parent_comment_id"

  validates :content, presence: true, length: {minimum: 1, maximum: 40000}
  validates :user_id, presence: true
  validates :post_id, presence: true
end
```

### Model Spec

For the model spec it will be the same as topic and post except for the requirement of testing both of our factories.

*spec/models/comment_spec.rb*
```ruby
require 'rails_helper'

RSpec.describe Comment, type: :model do
  it "has a valid factory" do
    expect(FactoryGirl.build(:comment)).to be_valid
    expect(FactoryGirl.build(:child_comment)).to be_valid
  end
end
```

this gives us the error:
```irb
     Failure/Error: expect(FactoryGirl.build(:comment)).to be_valid

     NoMethodError:
       undefined method `content=' for #<Comment:0x0000000a2da488>
       Did you mean?  concern
```

The new factory isn't realizing that the content attribute has been updated.  This is because we changed the value in our database and haven't updated the test database to reflect that.  Run the following command:

```irb
rails db:test:prepare
```

Now we are faced with the following
```irb
     Failure/Error: expect(FactoryGirl.build(:comment)).to be_valid
       expected #<Comment id: nil, content: "Qui qui vitae sunt nam quia iure. Explicabo dolor ...", post_id: 1, user_id: 1, parent_comment_id: nil> to be valid, but got errors: Parent comment must exist
```

We only want the parent to be optional.  I found [this stackoverflow post](http://stackoverflow.com/questions/16699877/rails-optional-belongs-to) which states that in rails 5 we can have optional belongs_to.  Lets go ahead and add it to our model

```ruby
belongs_to :parent_comment, :class_name => "Comment", :foreign_key => "parent_comment_id", optional: true
```

For some reason this also locked the database.  Go ahead and run the command from the previous post

```irb
rails console
ActiveRecord::Base.connection.execute("BEGIN TRANSACTION; END;")
exit
```

Now all the tests pass and we can go ahead to creating the comments in the database.

### Creating Comments

Pseudo code for creating a comment
- User logs in and clicks on the forum link in the nav bar
- User clicks on a topic then clicks on a post within that topic
- User clicks on the original post or a comment to that post and clicks reply
- User fills in the text area and clicks submit
- User is redirected to the url of the post and sees their comment below the post or comment they replied to

Some notes for the following tests
- We have to create a parent comment ahead of time so that we have a reply to comment button needed to create a child comment
- I am reducing the amount of times the test checks for invalid inputs to just once since it is only there to test that the errors are implemented
- I want the routes set up so that "/topics/#{@topic.id}/posts/#{@post.id} shows all of the direct child comments to the post
- The route "/topics/#{@topic.id}/posts/#{@post.id}/comments/#{@comment.id} to show the parent post, but only the first parent comment
- trying to create a comment as a guest through the 2 different paths ensures a guest can see all of the website content until they try to make a new comment

*spec/features/posts/create_comment_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Creating a Comment" do
  before do
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic)
    @parent_comment = FactoryGirl.create(:comment)
    @comment = FactoryGirl.build(:comment)
    @child_comment = FactoryGirl.build(:child_comment)
  end

  feature "as a user" do
    before do
      login_as(@user)
      visit "/"
      click_link "Forum"
      click_link "#{@topic.name}"
      click_link "#{@post.title}"
    end

    feature "in reply to a post" do
      before do
        click_link "Reply to Post"
      end

      scenario "with valid inputs" do
        fill_in "Content", with: @comment.content
        click_button "Create Comment"
        expect(page).to have_content("Your comment has been successfully created")
        expect(page).to have_content(@post.title)
        expect(page).to have_content(@comment.content)
      end

      scenario "with invalid inputs" do
        fill_in "Content", with: ""
        click_button "Create Comment"
        expect(page).to have_content("Content can't be blank")
      end
    end

    feature "in reply to a comment" do
      scenario "with valid inputs" do
        click_link "Reply to Comment"
        fill_in "Content", with: @child_comment.content
        click_button "Create Comment"
        expect(page).to have_content(@post.title)
        expect(page).to have_content(@parent_comment.content)
        expect(page).to have_content(@child_comment.content)
      end
    end
  end

  feature "as a guest" do
    scenario "through the user interface" do
      visit "/"
      click_link "Forum"
      click_link "#{@topic.name}"
      click_link "#{@post.title}"
      click_link "Reply to Post"
      expect(page).to have_content("You must sign in or sign up to view this page")
      expect(current_path).to eq(new_user_registration_path)
    end

    scenario "by going directly to the route" do
      visit "/topics/#{@topic.id}/posts/#{@post.id}/comments/new"
      expect(page).to have_content("You must sign in or sign up to view this page")
      expect(current_path).to eq(new_user_registration_path)
    end
  end
end
```

Looking at the first error:
```irb
Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: click_link "#{@post.title}"

     Capybara::ElementNotFound:
       Unable to find link "Omnis quibusdam ut illo dolores sed."
```

After clicking on a topic we are unable to see the list of all posts under that topic.  So we need to add it to the show view for topics.  The topic will not always have posts so we need to add a check to see if the topics has any posts with the .blank? method.  In rails this covers if the object is nil, false, empty, or whitespace.  Then we will loop through each post under that topic and create a link leading to the post.  If there are no posts, we will state that there are no posts under the topic.

*app/views/topics/show.html.erb*
```erb
<% unless @topic.posts.blank? %>
  <% @topic.posts.each do |post| %>
    <div class="row">
      <%= link_to post.title, topic_post_path(post) %>
    </div>
  <% end %>
<% else %>
  <p>There are no posts under this topic</p>
<% end %>

<%= link_to "New Post", new_topic_post_path(@topic), class: "btn btn-primary" %>
```

After creating that file we are still unable to find the post link after clicking on the topic within the test.  After putting a debugger between clicking the topic link and clicking the post link, I found that the two are unrelated.  [I found this link](https://github.com/thoughtbot/factory_girl/issues/683) which explained how to link objects within the create method.  Just change @post = FactoryGirl.create(:post) to @post = FactoryGirl.create(:post, topic: @topic).

Moving on to the next error

```irb
  1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: click_link "Reply to Post"

     Capybara::ElementNotFound:
       Unable to find link "Reply to Post"
```

The user has now clicked on a topic and clicked on the post owned by the topic.  Now within the show view for posts we need to create the reply to post button which links to the new comment path.  The comment path is going to be embedded within the post path which in itself is embedded within the topic path.  The route will require both the topic id and the post id within the parameters.

*app/views/posts/show.html.erb*
```erb
<%= link_to "Reply to Post", new_topic_post_comment_path(@post.topic, @post), class: "btn btn-primary" %>
```

The next error is going to be that this route is not defined, so we can go ahead and add it to routes.rb.  Like posts, we will be performing all actions on comments except for indexing them.  The comments route will go within the do block of posts to make the following.

*config/routes.rb*
```ruby
  resources :topics, only: [:create, :new, :index, :show] do
    resources :posts, except: [:index] do
      resources :comments, except: [:index]
    end
  end
```

The next error saying that the comments controller is missing indicates that the route is working fine.

```irb
uninitialized constant CommentsController
```

We can go ahead and create the new, create, and show actions as the bare minimum to create a comment.  The parameters needed to create a new comment is content, post_id, user_id, and optionally parent_comment id.  the post_id will be found using the parameter in the route and user_id found with current_user.  At the moment I'm not sure how to find the parent comment id and will go back to it when we get an error.  After creating a comment we will redirect to the comment path which needs the topic id and post id.  Both of which are found in the parameters of the url.

*app/controllers/comments_controller.rb*
```ruby
class CommentsController < ApplicationController

  def new
    @comment = Comment.new
  end

  def create
    @comment = Comment.new(comment_params)
    @topic = Topic.find(params[:topic_id])
    @post = Post.find(params[:post_id])
    @comment.user = current_user
    @comment.post = @post
    if @comment.save
      flash[:success] = "Your comment has been successfully created"
      redirect_to topic_post_comment_path(@topic, @post, @comment)
    else
      render 'new'
    end
  end

  def show
    @comment = Comment.find(params[:id])
  end

  def comment_params
    params.require(:comment).permit(:content)
  end
end
```

Next error saying the new template is missing

```irb
CommentsController#new is missing a template for this request format and variant.
```

Then that it can't find the content field

```irb
  1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: fill_in "content", with: @comment.content

     Capybara::ElementNotFound:
       Unable to find field "content"
```

Now we can fill in the field and error for comments.  Like with posts we will eventually want to edit comments, and will put everything into a form partial

*app/views/comments/_form.html.erb*
```erb
<%= render 'shared/errors', obj: @comment %>
<%= form_for([@topic, @post, @comment], url: topic_post_comments_path) do |f| %>
  <div class="form-group">
    <%= f.label :content %>
    <%= f.text_area :content, class: "form-control" %>
  </div>
  <div class="form-group">
    <%= f.submit class: "btn btn-primary" %>
  </div>
<% end %>
```

*app/views/comments/new.html.erb*
```erb
<h2>Create a new Comment</h2>
<%= render 'form' %>
```

We now get an error saying the show template is missing, indicating that everything worked out for creating the comment

```irb
CommentsController#show is missing a template for this request format and variant.
```

With the show template in place we get the following

```irb
  1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: expect(page).to have_content(@post.title)
```

Once we reply to a post we want to be redirected to a page which shows the comment we just created as well as the post above.  To keep things organized, we can render the post show template within the comment show view.  However for this to work the show view within the posts folder needs to be a parital.  Rename app/views/posts/show.html.erb to app/views/posts/_show.html.erb.  Then create another file linking to the show partial in the posts folder.

*app/views/posts/show.html.erb*
```erb
<%= render 'show' %>
```

The comments view will be rendering the same thing for now, but it needs to specify the folder since the partial is not in the same directory

*app/views/comments/show.html.erb*
```erb
<%= render 'posts/show' %>
```

This gives the following error:

```irb
  1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: <h2><%= @post.title %></h2>

     ActionView::Template::Error:
       undefined method `title' for nil:NilClass
```

Since it is the comments controller rendering this view, we need to add the instance variable @post that we are rendering in the view.

*app/controllers/comments_controller.rb*
```ruby
  def show
    @comment = Comment.find(params[:id])
    @post = @comment.post
  end
```

Which gives the following

```irb
1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: expect(page).to have_content(@comment.content)
```

For now we can just put it inside a div and worry about the styling later on.

*app/views/comments/show.html.erb*
```erb
<div>
  <%= @comment.content %>
</div>
```

Now we get the first error for replying to a comment.

```irb
  1) Creating a Comment as a user in reply to a comment with valid inputs
     Failure/Error: click_link "Reply to Comment"

     Capybara::ElementNotFound:
       Unable to find link "Reply to Comment"
```

The route for this button will be the same as creating a comment in reply to a post.  However we need to set a parameter parent_comment_id equal to the id of the comment we are replying to.

*app/views/comments/show.html.erb*
```erb
<%= render 'posts/show' %>
<div>
  <%= @comment.content %>
  <%= link_to "Reply to Comment", new_topic_post_comment_path(@post.topic, @post, parent_comment_id: @comment.id),
                                  class: "btn btn-primary btn-xs" %>
</div>
```

Now we still have the error.  However that is because in the test we are clicking on the topic link, then the post link, and then clicking reply to comment for the comment that should be listed underneath the post.  We only saw the content of the comment because we redirected to the comment path after creating it.  So we need to add the list of comments that belong to the post to the post show page.  We will be rendering the comment show page multiple times, so go ahead and rename it to app/views/comments/_show.html.erb.  Then we need to create a file to replace the show file which will render the posts show page.

*app/views/comments/show.html.erb*
```erb
<%= render '/posts/show' %>
```

This requires the posts show page to be a partial, so you can rename it to app/views/posts/_show.html.erb.  Then we need to link to it from a new show file

*app/views/posts/show.html.erb*
```erb
<%= render 'show' %>
```

In addition, the comments we are making to other comments are currently not having their parent comment id's set.  In the comment form, we need to add a hidden field which takes in the parameter set by the button to create the new comment

*app/views/comments/_form.html.erb*
```erb
<%= f.hidden_field(:parent_comment_id, :value => params[:parent_comment_id]) %>
```

Then since we are adding this through the form, we need to accept it in our controller as a parameter

*app/controllers/comments_controller.rb*
```ruby
params.require(:comment).permit(:content, :parent_comment_id)
```

Finally we get to the show form.  The functionality we want is if the @comment instance variable exists, then that means the file is being rendered from the comment controller and we want to focus on an individual comment and its parent for context if it exists.  So we need an if statement for the instance variable, then within thate statement check if a parent comment exists and render it before the selected comment.  The selected comment will also be placed within a div with class highlight.  For now the class will just have a style of background:yellow.

Otherwise the page is being rendered by the post controller, and we want to list each comment belonging to the post.  However we want to first put down comments with no parent comments since that means it replied directly to the post.  Then for each of those comments, check if there are any child comments and render those.  Looping through all comments without a parent and then all coments with children will eventually display all the comments belonging to the post.

*app/views/posts/_show.html.erb*
```erb
<h2><%= @post.title %></h2>
<%= @post.content %>
<%= link_to "Reply to Post", new_topic_post_comment_path(@post.topic, @post), class: "btn btn-primary" %>

<% if @comment %>
  <% if @comment.parent_comment %>
    <%= render '/comments/show', comment: @comment.parent_comment %>
  <% end %>
  <div class="highlight">
    <%= render '/comments/show', comment: @comment %>
  </div>
<% else %>
  <% @post.comments.each do |comment| %>
    <% if comment.parent_comment.nil? %>
      <%= render '/comments/show', comment: comment %>
    <% end %>
    <% unless (comment.child_comments.blank?) %>
      <% comment.child_comments.each do |child_comment| %>
        <%= render '/comments/show', comment: child_comment %>
      <% end %>
    <% end %>
  <% end %>
<% end %>
```

We still get the error, but that is because I forgot once again to add the post relationship to the parent comment in the spec.  Change the parent_comment line to the following:

*app/spec/comments/create_comment_spec.rb*
```ruby
@parent_comment = FactoryGirl.create(:comment, post: @post)
```

Finally the last 2 errors are just the lack of authentication in the controller

```irb
  1) Creating a Comment as a guest through the user interface
     Failure/Error: expect(page).to have_content("You must sign in or sign up to view this page")
       expected to find text "You must sign in or sign up to view this page" in "Prostate and Cholesterol Forum Sign in Sign up Create a new Comment Content"
     # ./spec/features/comments/create_comment_spec.rb:61:in `block (3 levels) in <top (required)>'

  2) Creating a Comment as a guest by going directly to the route
     Failure/Error: expect(page).to have_content("You must sign in or sign up to view this page")
       expected to find text "You must sign in or sign up to view this page" in "Prostate and Cholesterol Forum Sign in Sign up Create a new Comment Content"
     # ./spec/features/comments/create_comment_spec.rb:67:in `block (3 levels) in <top (required)>'
```

add the authenticate_user methos to the controller except for the show action

*app/controllers/comments_controller.rb*
```ruby
before_action :authenticate_user!, except: [:show]
```

Now we can move on to editing the models [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-3---editing-the-models/)
