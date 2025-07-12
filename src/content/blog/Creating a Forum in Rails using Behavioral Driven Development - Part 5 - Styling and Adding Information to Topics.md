---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 5 - Styling and Adding Information to Topics"
description: "Styling topics for a forum in Rails"
pubDate: "May 17 2017"
heroImage: "/blog-placeholder-2.jpg"
---

Let's start by adding to and styling the topics listing page and work our way down.

Things I would like to add to the topics listing

- The number of posts belonging to the topic
- A link to the latest post created in the topic as well as the user who created that post and how long ago it was created

In order to access how long ago posts were created, we will have to add timestamps to the models.  We aren't using all of them at once, but it is generally best practice to add timestamps to every table in the database and we might need to access them in the future.

```irb
rails g migration add_timestamps_to_topics_posts_comments
```

I tried `add_timestamps :topics` but would get the error

```irb
SQLite3::SQLException: Cannot add a NOT NULL column with default value NULL: ALTER TABLE "topics" ADD "created_at" datetime NOT NULL
```

When creating timestamps rails automatically sets the propery null:false.  However from [this stackoveflow post](http://stackoverflow.com/questions/3170634/how-to-solve-cannot-add-a-not-null-column-with-default-value-null-in-sqlite3) they explain that in sqlite you can't add a column with null: false to a table which is already created without a default.  We don't want to add a default to our timestamps as it doesn't really make sense.  .

So we will create the created_at and updated_at columns manually, then change the column to have :null => false right after it is created as sqlite doesn't care if you change a property to null without a default.

*generated migration file*
```ruby
class AddTimestampsToTopicsPostsComments < ActiveRecord::Migration[5.0]
  def self.up
    add_column :topics, :created_at, :datetime
    change_column :topics, :created_at, :datetime, :null => false
    add_column :topics, :updated_at, :datetime
    change_column :topics, :updated_at, :datetime, :null => false

    add_column :posts, :created_at, :datetime
    change_column :posts, :created_at, :datetime, :null => false
    add_column :posts, :updated_at, :datetime
    change_column :posts, :updated_at, :datetime, :null => false

    add_column :comments, :created_at, :datetime
    change_column :comments, :created_at, :datetime, :null => false
    add_column :comments, :updated_at, :datetime
    change_column :comments, :updated_at, :datetime, :null => false
  end

  def self.down
    remove_column :topics, :created_at
    remove_column :topics, :updated_at

    remove_column :posts, :created_at
    remove_column :posts, :updated_at

    remove_column :comments, :created_at
    remove_column :comments, :updated_at
  end
end
```

Then we will clean the database as we can't have any records without a timestamp and run the migration

```irb
rails console
Topic.delete_all
Post.delete_all
Comment.delete_all
end
rails db:migrate
rails db:test:prepare
```

Each topic will go into a div and we'll put the title, number of posts, and link to latest post into paragraphs within the div.  The div will have a class of topics so we can easily style this page in the scss file.  We will use the pluralize function on the post count to have it say post if there is only 1 post, and say posts for a count above one.  When displaying the last post we need to run a check to see if the topic.posts call is empty as otherwise we will be looking for a title for a model that doesn't exist.

Putting it all into a spec

*spec/features/topics/list_topics_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Listing Topics" do
  before do
    @admin = FactoryGirl.create(:admin)
    @topic1 = FactoryGirl.create(:topic)
    @topic2 = FactoryGirl.create(:topic)
    @post1 = FactoryGirl.create(:post, user: @admin, topic: @topic1)
  end

  scenario "shows all topics" do
    visit "/"
    click_link "Forum"
    expect(page).to have_content(@topic1.name)
    expect(page).to have_content(@topic2.name)
    expect(page).to have_content("0 posts")
    expect(page).to have_content("1 post")
    expect(page).to have_content("last post: #{@post1.title} by #{@admin.username} less than a minute ago")
  end
end
```

which gives the following

```irb
  3) Listing Topics shows all topics
     Failure/Error: expect(page).to have_content("0 posts")
       expected to find text "0 posts" in "Prostate and Cholesterol Forum Sign in Sign up dignissimos corrupti"
     # ./spec/features/topics/list_topics_spec.rb:16:in `block (2 levels) in <top (required)>'
```

Let's add div to the index file and put the pluralize function in a p tag and the topic link in an h2 tag

*app/views/topics/index.html.erb*
```erb
  <div class="topics">
    <h2><%= link_to "#{topic.name}", topic_path(topic) %></h2>
    <p><%= pluralize(topic.posts.count, "post") %></p>
  </div>
```

Now we have the following error:

```irb
  1) Listing Topics shows all topics
     Failure/Error: expect(page).to have_content("last post: #{@post1.title} by #{@admin.username} less than a minute ago")
```
Let's add the link to the last post, the link to the user who created the post, and the time_ago_in_words function on when the post was created

*app/views/topics/index.html.erb*
```erb
    <p>
      last post: <%= link_to "#{topic.posts.last.title}", topic_post_path(topic, topic.posts.last) %>
      by <%= link_to "#{topic.posts.last.user.username}", profile_path(topic.posts.last.user.profile) %>
      <%= time_ago_in_words(topic.posts.last.created_at) %> ago
    </p>
```

Which gives us:

```irb
  5) Listing Topics shows all topics
     Failure/Error: last post: <%= link_to "#{topic.posts.last.title}", topic_post_path(topic, topic.posts.last) %>

     ActionView::Template::Error:
       undefined method `title' for nil:NilClass
```

We didn't surround that parageraph in the check to see if the topic had any posts.  We put in topic2 into the test to check for this functionality.  Let's add the check and put it all together.

*app/views/topics/index.html.erb*
```erb
<% @topics.each do |topic| %>
  <div class="topics">
    <h2><%= link_to "#{topic.name}", topic_path(topic) %></h2>
    <p><%= pluralize(topic.posts.count, "post") %></p>
    <% unless topic.posts.empty? %>
      <p>
        last post: <%= link_to "#{topic.posts.last.title}", topic_post_path(topic, topic.posts.last) %>
        by <%= link_to "#{topic.posts.last.user.username}", profile_path(topic.posts.last.user.profile) %>
        <%= time_ago_in_words(topic.posts.last.created_at) %> ago
      </p>
    <% end %>
  </div>
<% end %>

<% if current_user.try(:admin?) %>
  <%= link_to "New Topic", new_topic_path, class: "btn btn-primary" %>
<% end %>
```

Now we have 0 errors, and can move on to adding css.  To seperate the topics let's add a border-bottom 1px wide with a light grey color:

*app/stylesheets/stylesheets/forum.scss*
```css
/* styling for forum models topics, posts, and comments */

.topics {
	border-bottom: 1px solid #d1d1d1;
}
```

To lower the emphasis on the number of posts and last posts, let's make their font color a dark grey and set their font-size to small.  Put the following inside the .topics selector

*app/stylesheets/stylesheets/forum.scss*
```css
	p {
	  color: #888;
	  font-size: small;
	}
```

I'm happy enough with this styling for now, so we can move on to styling the actual post page [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-6---styling-and-adding-information-to-posts-listing-topic-page/)
