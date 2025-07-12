---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 8 - Styling and Adding Information to Comments Listing (post) page"
description: "Styling comments for a forum in Rails"
pubDate: "Jun 8 2017"
heroImage: "/blog-placeholder-2.jpg"
---

The post page is comprised of 2 halfs.  The content of the post and the listing of comments belonging to the post.  We just finished styling the post in both the listing page and then rendered out the content in a partial to use in the post page.  To finish up the basics of the forum the only thing we have left to do is to style and orgnaize the comments.

Right now all comments are listed directly underneath the post with no formatting, regardless of how long the chain of comments go they are styled as if they were direct children of the post.  We want to add a margin to each child in order to show it is in reply to the parent comment above it.  Due to this added margin, we will also have to add pseudo-pagination as the browser would eventually run out of room horizontally to fit in a very long chain of comments.  Let's set it up so that after 5 child comments there will be a direct link to the next child comment to make it the new start of the page; which then shows 5 more child comments and then another link to the next comment in the chain.

In addition to the pagination, we'll be adding the following information

- The username of the person who posted the comment and a link to their profile
- how long ago the comment was posted
- a link to the comment which would in turn highlight the comment and show all child comments up to 5 comments deep

For this test we want 7 comment instance variables.  Each comment will belong to the previous comment.  When viewing the post, we should see `@comment1` and every comment until `@comment6` since it is 5 comments deep.  In order to see `@comment7` the user will have to click a button "see additional replies" which will direct them to the comment page with `@comment7` as the highlighted comment.  I had to look up and use `instance_variable_set` and `instance_variable_get` in order to create all of the comments in a .each loop as the variables names are dynamic without creating 6 comments manually. Putting this into a test:

*spec/features/comments/listing_comments_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Listing Comments" do
  before do
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic)
    @comment1 = FactoryGirl.create(:comment, post: @post, user: @user)
    (1..6).each do |i|
      instance_variable_set("@comment" + "#{i+1}", FactoryGirl.create(:child_comment,
      parent_comment: instance_variable_get("@comment" + "#{i}"), post: @post))
    end
    visit "/"
    click_link "Forum"
    click_link @topic.name
    click_link @post.title
  end

  scenario "shows up to 5 child comments deep" do
    expect(page).to have_content(@comment1.content)
    expect(page).not_to have_content(@comment7.content)
    click_link "see additional replies"
    expect(page).not_to have_content(@comment1.content)
    expect(page).to have_content(@comment7.content)
  end

  scenario "shows basic information for each comment" do
    expect(page).to have_content("#{@user.username} - less than a minute ago")
  end
end
```

We get our first error with `@comment7` which means our loop worked properly

```irb
  1) Listing Comments shows up to 5 child comments deep
     Failure/Error: expect(page).not_to have_content(@comment7.content)
```

To start this off, we need to change the logic in how we loop through the comments belonging to the post.  At the moment we loop through every single comment and dislay it if it has no parent comments (meaning it is in direct reply to the post).  Then if it has children, it will loop through each child comment and display it.  Just as a refresher here is our current logic:

```erb
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
```

To clean this up and alter it to our needs lets put some of the logic into our models.  Starting with grabbing all direct replies to the post.  We will use .select on the comments with the same conditional we used in the view of .parent_comment.nil?

*app/models/post.rb*
```ruby
  def direct_replies
    self.comments.select { |comment| comment.parent_comment.nil? }
  end
```

Now for comments we want to calculate the amount of ancestors each comment has.  In other words how many times you would have to call .parent_comment on a comment before it is nil as is is a direct descendant of the original post.  Knowing this we can find out the differences in ancestors between a selected comment and another comment and know whether to display it if the difference is less than 5.  For this we have to use a recursive function.  I got a quick refresher from [this stackoverflow post](https://stackoverflow.com/questions/6418017/what-is-recursion-and-how-does-it-work).

We will start off with the function to be called initially on the comment object.  This function will call the recursive function on each successive parent comment.  We use self to refer to the comment the function is being called on.  We need an instance variable as we don't want to reinitialize the variable between function calls.

The ancestors instance variable will be set by using the ||= operator.  I'm not an expert on the subject, but what is essentially does is if a value exists for the @ancestors instance variable, it will use the already exisiting value.  However if there is no value, then it will set it to 0.  So @ancestors equals @ancestors or it equals 0, hence the equals or syntax.  During the first pass, it will be set to 0.

```ruby
  def ancestor_count
    @ancestors ||= get_ancestors_count(self)
  end
```

In the get_ancestors_count function, we are passing in a variable called comment which will hold the initial comment passed in by self, and then the parent of that comment until there are not more parents

```ruby
def get_ancestors_count(comment)
```

We need to use the ||= operator to set the @ancestors instance variable within the function to make sure it won't be set to 0 on every pass, just the first one.

```ruby
@ancestors ||= 0
```

Now if the comment passed into the function has no parent comments, then we want to return @ancestors.  Otherwise, we want to increment it by 1 and then call the function again on the comments parent comment.  The function then ends up looking like this:

```ruby
  def get_ancestors_count(comment)
    @ancestors ||= 0

    if comment.parent_comment.nil?
      return @ancestors
    else
      @ancestors += 1
      get_ancestors_count(comment.parent_comment)
    end
  end
```

Now that we have access to the amount of ancestors a comment has, we need to find all the children of a comment within 5 levels of children comments deep.  You can think of this as a tree with the comment we are looking at as the root, and we want to find all comments within 5 branches of the root of the tree.  We might not always want 5, so we can use that as a parameter for the function.  We also want access to the root comment throughout all of the recursion, so we will set an @root instance variable equal to self.

The @children_tree instance variable will be an array of comments where instead of adding 1 if the function doesn't return, we will simply add a comment to the array.  Putting this into a function called children_tree:

```ruby
  def children_tree(depth)
    @root = self
    @children_tree ||= get_children_tree(self, depth)
  end
```

 This function will start off with setting the @children_tree instance variable to an empty array with the or equals operator just like the previous function

```ruby
  def get_children_tree(comment, depth)
  @children_tree ||= []
```

Now we have to iterate through each child_comment of the passed in comment.  If it is too far from the @root, then we will return the @children_tree array as is.  If it is within the specified depth, then we will add it to the array and call the function again with the child_comment

```ruby
    comment.child_comments.each do |child_comment|
      if (child_comment.ancestor_count - @root.ancestor_count) > depth
        return @children_tree
      else
        @children_tree << child_comment
        get_children_tree(child_comment, depth)
      end
    end
```

Finally if the function has finished iterating through each child comment and all of them were within the depth, then we will return the entire array after the child_comments loop.  The function ends up looking like this:

```ruby
  def get_children_tree(comment, depth)
  @children_tree ||= []

    comment.child_comments.each do |child_comment|
      if (child_comment.ancestor_count - @root.ancestor_count) > depth
        return @children_tree
      else
        @children_tree << child_comment
        get_children_tree(child_comment, depth)
      end
    end
    return @children_tree
  end
```

And finally everything we just added put together:

*app/models/comment.rb*
```ruby
  def get_ancestors_count(comment)
    @ancestors ||= 0

    if comment.parent_comment.nil?
      return @ancestors
    else
      @ancestors += 1
      get_ancestors_count(comment.parent_comment)
    end
  end

  def get_children_tree(comment, depth)
  @children_tree ||= []

    comment.child_comments.each do |child_comment|
      if (child_comment.ancestor_count - @root.ancestor_count) > depth
        return @children_tree
      else
        @children_tree << child_comment
        get_children_tree(child_comment, depth)
      end
    end
    return @children_tree
  end

  def ancestor_count
    @ancestors ||= get_ancestors_count(self)
  end

  def children_tree(depth)
    @root = self
    @children_tree ||= get_children_tree(self, depth)
  end
end
```

Now in the show partial instead of iterating through child_comments, we can iterate through the children_tree.  When rendering the direct comment, we will render a tree 5 levels deep.  When rendering a selected comment, we only need to render a tree 4 levels deep as the parent comment is showing as well.  We no longer need a check to see if the array is empty as the function will just give a empty array which won't be looped over instead of giving an undefined error.

*app/views/posts/_show.html.erb*
```erb
<% if @comment %>
  <% if @comment.parent_comment %>
    <%= render '/comments/show', comment: @comment.parent_comment %>
  <% end %>
  <div class="highlight">
    <%= render '/comments/show', comment: @comment %>
  </div>
  <% @comment.children_tree(4).each do |child_comment| %>
    <%= render '/comments/show', comment: child_comment %>
  <% end %>
<% else %>
  <% @post.direct_replies.each do |comment| %>
    <%= render '/comments/show', comment: comment %>
    <% comment.children_tree(5).each do |child_comment| %>
      <%= render '/comments/show', comment: child_comment %>
    <% end %>
  <% end %>
<% end %>
```

This passes the first error and leaves us with the following:

```irb
  1) Listing Comments shows up to 5 child comments deep
     Failure/Error: click_link "see additional replies"

     Capybara::ElementNotFound:
       Unable to find link "see additional replies"
```

Next we need to add logic that detects if the last child in the tree has any child comments of its own which is not being displayed.  If it does, we will add the link called see additional replies and have it link to the last child in the tree.  When renderig a post, this means seeing if the number of ancestors is >= 5, and when displaying a comment it means checking if the difference between the comment and child comment is greater than 4.

For the post display:

```erb
        <% if (child_comment.ancestor_count >= 5 && !child_comment.child_comments.empty?) %>
          <%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment) %>
        <% end %>
```

and for the comment display:

```erb
    <% if ((child_comment.ancestor_count - @comment.ancestor_count) >= 4 && !child_comment.child_comments.empty?) %>
      <%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment) %>
    <% end %>
```

This again is probably too much logic for a view, so let's extract it into the comment model with passing in the ancestor and the depth.  This function is a conditional which will return true or false on its own.

```ruby
  def has_hidden_replies(ancestor, depth)
    (self.ancestor_count - ancestor.ancestor_count) >= depth && !self.child_comments.empty?
  end
```

One more change we can make is to change the `app/views/comments/_show.html.erb` file into `app/views/comments/_comment.html.erb`.  This will save some syntax in rendering out the partial and follow better practices.  This turns `<%= render '/comments/show', comment: @comment %>` into `<%= render(@comment) %>`.  Implementing the has_hidden_replies function and the new partial gives the following:

*app/views/posts/_show.html.erb*
```erb
<h1><%= link_to @post.topic.name, topic_path(@post.topic) %></h1>

<%= render(@post) %>

<% if @comment %>
  <% if @comment.parent_comment %>
    <%= render(@comment.parent_comment) %>
  <% end %>
  <div class="highlight">
    <%= render(@comment) %>
  </div>
  <% @comment.children_tree(4).each do |child_comment| %>
    <%= render(child_comment) %>
    <% if child_comment.has_hidden_replies(@comment, 4) %>
      <%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment) %>
    <% end %>
  <% end %>
<% else %>
  <% @post.direct_replies.each do |comment| %>
    <%= render(comment) %>
    <% comment.children_tree(5).each do |child_comment| %>
      <%= render(child_comment) %>
      <% if child_comment.has_hidden_replies(comment, 5) %>
        <%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment) %>
      <% end %>
    <% end %>
  <% end %>
<% end %>
```

This gives us our last and easy to tackle error:

```irb
  2) Listing Comments shows basic information for each comment
     Failure/Error: expect(page).to have_content("#{@user.username} - less than a minute ago")
```

Let's go ahead and add that line to the comment partial along with a few other changes.   We'll change the buttons to use the info and danger class to differentiate them, we'll wrap the content in a paragraph, and we'll add the class comment to the div in order to style it later on.

*app/views/comments/_comment.html.erb*
```erb
<div class="comment">
  <p>
    <%= link_to "#{comment.user.username}", profile_path(comment.user.profile) %> -
    <%= time_ago_in_words(comment.created_at) %> ago
  </p>
  <p><%= comment.content %></p>
  <%= link_to "Reply to Comment", new_topic_post_comment_path(@post.topic, @post, :parent_comment_id => comment.id),
                                  class: "btn btn-primary btn-xs" %>
  <% if comment.user == current_user && !comment.deleted %>
    <%= link_to "edit comment", edit_topic_post_comment_path(@post.topic, @post, comment),
                                class: "btn btn-info btn-xs" %>
    <%= link_to "delete comment", delete_topic_post_comment_path(@post.topic, @post, comment),
                                  method: :patch, data: { confirm: "Are you sure?" },
                                  class: "btn btn-danger btn-xs" %>
  <% end %>
</div>
```

One last change is that we are missing some information in our test.  We only tested the first layer of "see additional replies" without testing if it would exist when viewing a comment directly.  Let's write a test visiting the url of comment 2 in which we should see 4 comments deep (comment 2 through 6) but not comment 7.  Then click "see additional replies" and we should see comment 7 but not 2.

Start off with removing the following lines from app/views/posts/_show.html.erb

```erb
    <% if child_comment.has_hidden_replies(@comment, 4) %>
      <%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment) %>
    <% end %>
```

All the tests should still pass since we weren't testing for this functionality.  Now add the following test to our listing_comments_spec

```ruby
  scenario "shows up to 4 child comments deep when viewing a comment" do
    visit "/topics/#{@topic.id}/posts/#{@post.id}/comments/#{@comment2.id}"
    expect(page).to have_content(@comment2.content)
    expect(page).not_to have_content(@comment7.content)
    click_link "see additional replies"
    expect(page).not_to have_content(@comment2.content)
    expect(page).to have_content(@comment7.content)
  end
```

You should see the following

```irb
  1) Listing Comments shows up to 4 child comments deep when viewing a comment
     Failure/Error: click_link "see additional replies"

     Capybara::ElementNotFound:
       Unable to find link "see additional replies"
```

Now go ahead and add the lines back in, the test will pass and we can move onto styling.

At the moment each comment is listed right underneath the previous and you can't tell whether a comment is in reply to the post or the previous comment.  In order to distinguish, we will add a margin-left multiplied by the number of levels deep the comment is relative to the root comment of the page (or the post).  We can do this by adding a dynamic class in addition to the class of comment.  We will assign margin-level-0 to the parent of a selected comment, or to a direct reply to the post.  Then we will add 1 level for each reply.

If @comment and @comment.parent_comment exists (meaning the margin level has to be calculated relative to the @comment parent), we can calculate the margin-level with the following:

```ruby
comment.ancestor_count - @comment.parent_comment.ancestor_count
```
Otherwise (meaning we are viewing direct replies to the post), we can simply calculate the amount of ancestors a comment has.  DIrect replies will have 0 ancestors keeping with our assigned style.

```ruby
comment.ancestor_count
```

Putting this all together in the comment partial:

```ruby
<% if @comment && @comment.parent_comment %>
  <div class="comment margin-level-<%= comment.ancestor_count - @comment.parent_comment.ancestor_count %>">
<% else %>
  <div class="comment margin-level-<%= comment.ancestor_count %>">
<% end %>
```

The largest margin-level that can be evaluated is 5 since we are only showing a maximum of 5 comments deep in our child tree.  So let's apply the margins to these class in css using %width of the screen:

```scss
.margin-level-0 {
	margin-left: 5%;
}

.margin-level-1 {
	margin-left: 15%;
}

.margin-level-2 {
	margin-left: 20%;
}

.margin-level-3 {
	margin-left: 25%;
}

.margin-level-4 {
	margin-left: 30%;
}

.margin-level-5 {
	margin-left: 35%;
}
```

We also need to apply margin-level-5 to the "see additional replies" link to have it underneath the comment it is referring to

```erb
<%= link_to "see additional replies", topic_post_comment_path(@post.topic, @post, child_comment), class: "margin-level-5" %>
```

Then in order to better keep track of comments and space them out let's make every second one have a light grey background color and add a 10px margin on the top and bottom:

```scss
.comment:nth-child(even) {
	background-color: #f4f4f9;
}

.comment {
	margin: 10px 0;
}
```
