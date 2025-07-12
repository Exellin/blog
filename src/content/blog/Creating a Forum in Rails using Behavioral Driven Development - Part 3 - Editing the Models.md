---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 3 - Editing the Models"
description: "Creating Topics and Posts for a forum in Rails"
pubDate: "May 2 2017"
heroImage: "/blog-placeholder-2.jpg"
---

First we need to look at what models and properties we want to have the ability to edit.  For Topics which only an administrator can create they should not be editable.  This is becauase if there is a significant amount of content underneath the topic and it is changed, none of the posts underneath may be relevant or make sense.

## Editing Posts

Pseudo code for editing a post
- User logs in
- User clicks forum link
- User clicks on a topic he previously made a post on
- User clicks on the title of the post he previously created
- User only sees the button "edit post" since he was the original creator and clicks on it
- User redirected to edit page and edits the content and/or title of their post
- User clicks edit post
- User redirected to post where they see the edited content

Putting it all into a spec

*spec/posts/edit_post_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Editing a Post" do
  before do
    @owner = FactoryGirl.create(:user)
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic, user: @owner)
  end

  scenario "as the owner of the post" do
    login_as(@owner)
    visit "/"
    click_link "Forum"
    click_link "#{@topic.name}"
    click_link "#{@post.title}"
    click_link "edit post"
    fill_in "Content", with: "edited content"
    click_button "Update Post"
    expect(page).to have_content("edited content")
    expect(page).to have_content("Post has been updated")
    expect(page).not_to have_content(@post.content)
  end

  feature "as another user" do
    before do
      login_as(@user)
    end

    scenario "through the user interface" do
      visit "/"
      click_link "Forum"
      click_link "#{@topic.name}"
      click_link "#{@post.title}"
      expect(page).not_to have_content("edit post")
    end

    scenario "by going directly to the route" do
      visit "/topics/#{@topic.id}/posts/#{@post.id}/edit"
      expect(page).to have_content("You can only edit or delete your own content")
      expect(current_path).to eq(topic_post_path(@topic, @post))
    end
  end
end
```

With little surprise the first error is that there is no "edit post" link

```irb
  1) Editing a Post as the owner of the post
     Failure/Error: click_link "edit post"

     Capybara::ElementNotFound:
       Unable to find link "edit post"
```

Add the following to the show post file underneath the reply to post link

*app/views/posts/_show.html.erb*
```erb
<% if @post.user == current_user %>
  <%= link_to "edit post", edit_topic_post_path(@post.topic, @post), class: "btn btn-xs btn-primary" %>
<% end %>
```

Nowe get that the action couldn't be found which means the link is working fine.

```irb
  1) Editing a Post as the owner of the post
     Failure/Error: click_link "edit post"

     AbstractController::ActionNotFound:
       The action 'edit' could not be found for PostsController
```

We can go ahead and add both the edit and the update actions.  In addition, we can set a set_post method to use for the edit, show, and update methods.  Add the following to the posts controller.

*app/controllers/posts_controller.rb*
```ruby
  before_action :set_post, only: [:edit, :show, :update]

  def edit
  end

  def update
    if @post.update(post_params)
      flash[:success] = "Post has been updated"
      redirect_to topic_post_path(@post.topic, @post)
    else
      render 'edit'
    end
  end
```

We now get an error saying the edit action is missing a template.  we already have a form from creating a post so it is very quick to add one.  Since the route wasn't working correctly for creating a post and we had to specify it, we will have to do the same for editing.  We will pass in a url variable from new.html.erb and edit.html.erb and specify it in the form.

*app/views/posts/edit.html.erb*
```erb
<h2>Edit Your Post</h2>
<%= render 'form', url: topic_post_path(@post) %>
```

We need to pass in the variable in new.html.erb as well.

*app/views/posts/new.html.erb*
```erb
<h2>Create a new Post</h2>
<%= render 'form', url: topic_posts_path %>
```

Then grab the variable in the form.

*app/views/posts/_form.html.erb*
```erb
<%= form_for([@topic, @post], url: url) do |f| %>
```

The only error now is allowing users to edit posts which they didn't create.  This means we didn't break the create action with passing the url.

```irb
  1) Editing a Post as another user by going directly to the route
     Failure/Error: expect(page).to have_content("You can only edit or delete your own content")
```

Let's create a private method called require_same_user which we will apply to the edit and update actions.  This needs to be set underneath :set_post so it will have access to the @post instance variable

*app/controllers/posts_controller.rb*
```ruby
  before_action :require_same_user, only: [:edit, :update]

  def require_same_user
    if current_user != @post.user
      flash[:danger] = "You can only edit or delete your own content"
      redirect_to topic_post_path(@post.topic, @post)
    end
  end
```

Now everything passes, and we can move on to editing comments.


## Editing Comments

The process for editing a comment is going to be almost exactly the same as editing a post.  In order to save time we won't be going through every single error and adding everything we need to to every file in one go.  Pseudo code for editing a comment

- User logs in
- User clicks forum link
- User clicks on a topic
- User clicks on a post he previously made a comment on
- User finds comment he made and clicks "edit comment"
- User redirected to edit page and edits the content of their comment
- User clicks edit comment
- User redirected to comment route where he sees the edited comment

Putting it all into a spec

*spec/comments/edit_comment_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Editing a Comment" do
  before do
    @owner = FactoryGirl.create(:user)
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic)
    @comment = FactoryGirl.create(:comment, post: @post, user: @owner)
  end

  scenario "as the owner of the comment" do
    login_as(@owner)
    visit "/"
    click_link "Forum"
    click_link "#{@topic.name}"
    click_link "#{@post.title}"
    click_link "edit comment"
    fill_in "Content", with: "edited content"
    click_button "Update Comment"
    expect(page).to have_content("edited content")
    expect(page).to have_content("Comment has been updated")
    expect(page).not_to have_content(@comment.content)
  end

  feature "as another user" do
    before do
      login_as(@user)
    end

    scenario "through the user interface" do
      visit "/"
      click_link("Forum")
      click_link "#{@topic.name}"
      click_link "#{@post.title}"
      expect(page).not_to have_content("edit comment")
    end

    scenario "by going directly to the route" do
      visit "/topics/#{@topic.id}/posts/#{@post.id}/comments/#{@comment.id}/edit"
      expect(page).to have_content("You can only edit or delete your own content")
      expect(current_path).to eq(topic_post_comment_path(@topic, @post, @comment))
    end
  end
end

```

This gives us an error saying it can't find the link "edit comment" which means are spec is set up correctly

```irb
  1) Editing a Comment as the owner of the comment
     Failure/Error: click_link "edit comment"

     Capybara::ElementNotFound:
       Unable to find link "edit comment"
```

 Like with post, we will do a quick check to see the if user the comment belongs to is the same as the current user.  If so, the edit comment button will show.

*app/views.comments/_show.html.erb*
```erb
  <% if comment.user == current_user %>
    <%= link_to "edit comment", edit_topic_post_comment_path(@post.topic, @post, comment),
                                class: "btn btn-primary btn-xs" %>
  <% end %>
```

Which gives the following error

```irb
  2) Editing a Comment as the owner of the comment
     Failure/Error: click_link "edit comment"

     AbstractController::ActionNotFound:
       The action 'edit' could not be found for CommentsController
```

We'll add the edit and update actions as well as the require_same_user and set_comment method so we don't have to come back to the controller right away if everything works out.

*app/controllers/comments_controller.rb*
```ruby
  before_action :set_comment, only: [:edit, :show, :update]
  before_action :require_same_user, only: [:edit, :update]

  def edit
  end

  def update
    if @comment.update(comment_params)
      @post = @comment.post
      @topic = @post.topic
      flash[:success] = "Comment has been updated"
      redirect_to topic_post_comment_path(@topic, @post, @comment)
    else
      render 'edit'
    end
  end

  def set_comment
    @comment = Comment.find(params[:id])
  end

  def require_same_user
    if current_user != @comment.user
      flash[:danger] = "You can only edit or delete your own content"
      redirect_to topic_post_comment_path(@comment.post.topic, @comment.post, @comment)
    end
  end
```

Which gives the following error:

```irb
  1) Editing a Comment as the owner of the comment
     Failure/Error: click_link "edit comment"

     ActionController::UnknownFormat:
       CommentsController#edit is missing a template for this request format and variant.
```

Like we did with posts, we will send the url from the new and edit file into the shared form file.

*app/views/comments/new.html.erb*
```erb
<h2>Create a new Comment</h2>
<%= render 'form', url: topic_post_comments_path %>
```

*app/views/comments/edit.html.erb*
```erb
<h2>Edit Your Comment</h2>
<%= render 'form', url: topic_post_comment_path(@comment) %>
```

*app/views/comments/_form.html.erb*
```erb
<%= form_for([@topic, @post, @comment], url: url) do |f| %>
```

Now everything passes, and we can move on to deleting the posts and comments [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-4---deleting-the-models/)
