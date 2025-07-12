---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 7 - Adding a Post partial"
description: "Creating a posts partial for a forum in Rails"
pubDate: "Jun 6 2017"
heroImage: "/blog-placeholder-2.jpg"
---

For this lesson we won't be writing any new tests as we aren't adding any new content.  We will be simply moving around content in files and relying on our tests already written to make sure we don't break anything in the process.

When on the post show page, we want to show all of the information about that post which was shown on the listing page. Therefore we should take the content shared by the two views into a _post partial.  However instead of the content being hidden behind a expand button, we want to show the content by default and we will require a conditional to differentiate between the two.  Cut the following from `app/views/topics/show.html.erb` and replace it with `<%= render(post) %>`.  Then paste it into the following file:

*app/views/posts/_post.html.erb*
```erb
<div class="post">
  <h2><%= link_to post.title, topic_post_path(@topic, post) %></h2>
  <p><%= pluralize(post.comments.count, "comment") %></p>
  <p>
    created <%= time_ago_in_words(post.created_at) %> ago by
    <%= link_to "#{post.user.username}", profile_path(post.user.profile) %>
  </p>
  <%= link_to "expand", topic_post_path(@topic, post), remote: true, class: "btn btn-default" %>
</div>
```

All tests should still pass.  Rails knows to render the post variable with the appropriate post partial file in the posts view directory.  Let's go ahead and replace everything outside of the @comment conditional in the `app/views/posts/_show` partial with `<%= render(@post) %>` and see what we have to fix.  We will also add a link to the topic while we are here.  Rails autmatically knows @post is the same as post in the partial and we don't have to specify.

*app/views/posts/_show.html.erb*
```erb
<h1><%= link_to @post.topic.name, topic_path(@post.topic) %></h1>

<%= render(@post) %>

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

This gives the following error:

```irb
     Failure/Error: <h2><%= link_to post.title, topic_post_path(@topic, post) %></h2>

     ActionView::Template::Error:
       No route matches {:action=>"show", :controller=>"posts", :id=>"1", :topic_id=>nil} missing required keys: [:topic_id]
```

The @topic variable isn't initialized through the posts_controller and comments_controller that this partial is being rendered by and therefore the route is invalid.  We could add it to the show action of those two controllers.  Or to keep the controllers clean we could replace the @topic in the link with post.topic.

This gives us the following:

```irb
     Failure/Error: <%= link_to "expand", topic_post_path(@topic, post), remote: true, class: "btn btn-default" %>

     ActionView::Template::Error:
       No route matches {:action=>"show", :controller=>"posts", :id=>"1", :topic_id=>nil} missing required keys: [:topic_id]
```

We only want to render this line if the partial is being controlled by the topic controller, so we can wrap it a conditional checking for the @topic instance variable.

Which gives the following:

```irb
  1) Creating a Comment as a user in reply to a post with valid inputs
     Failure/Error: click_link "Reply to Post"

     Capybara::ElementNotFound:
       Unable to find link "Reply to Post"
```

Now the tests will be complaining about missing the reply button, the edit button, and the delete button that the view is missing.  Since we are checking if the @topic instance variable exists when the view is controlled by the topics controller, we can add everything else we removed on the other end of the conditional for when the post and comment controllers are in charge.  One small change though is we can render the content partial instead of just @post.content.  Remember to specify the posts folder as the comments controller won't know where to find the partial otherwise.  I also changed all of the buttons to be extra small and changed the edit button to info and delete button to danger.

Putting this all together:

*app/views/posts/_post.html.erb*
```erb
<div class="post">
  <h2><%= link_to post.title, topic_post_path(post.topic, post) %></h2>
  <p><%= pluralize(post.comments.count, "comment") %></p>
  <p>
    created <%= time_ago_in_words(post.created_at) %> ago by
    <%= link_to "#{post.user.username}", profile_path(post.user.profile) %>
  </p>
  <% if @topic %>
    <%= link_to "expand", topic_post_path(@topic, post), remote: true, class: "btn btn-default" %>
  <% else %>
    <%= render 'posts/content' %>
    <%= link_to "Reply to Post", new_topic_post_comment_path(@post.topic, @post), class: "btn btn-primary btn-xs" %>
    <% if @post.user == current_user && !@post.deleted? %>
      <%= link_to "edit post", edit_topic_post_path(@post.topic, @post), class: "btn btn-info btn-xs" %>
      <%= link_to "delete post", delete_topic_post_path(@post.topic, @post),
                                 method: :patch, data: { confirm: "Are you sure?" },
                                 class: "btn btn-danger btn-xs" %>
    <% end %>
  <% end %>
</div>
```

In the next post, we are going to style and add information to comments [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-8---styling-and-adding-information-to-comments-listing-post-page/)
