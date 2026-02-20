---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 6 - Styling and Adding Information to Posts listing (topic) page"
description: "Styling posts for a forum in Rails"
pubDate: "Jun 1 2017"
---

Now we are ready to add styling and content to the posts listing (topic show) page.  This will be very similar to the topics listing page with the few differences being information about the topic the posts is in, and the metadata being about comments within the posts instead of posts within the topic.

Things I think are necessary when showing the posts listing page.  I didn't include buttons to comment directly to the post as I think the user should click on the post and see all other comments before they comment in order to avoid duplicates of the same comment.

- The Topic the posts are in as a header at the top of the page
- How long ago the post was created and a link to the user who created the post
- the number of comments belonging to the post
- a button to show the content of a post so you don't have to click on every post to see what they are about.  The user will click on the post only if they want to see comments or comment themselves.
- Some sort of pagination.  This wasn't necessary for topics as there will only be a few topics, but there could be hundreds or more posts in a topic.  The post will be sorted from newest to oldest.

To test the pagination, I need to use FactoryGirls create_list function to make many objects of the same class with a count equal to the amount of items displayed per page plus 1.  If we test that we can see each item on the page except for the one created first (as it is the oldest), then click on the next page and see only the oldest item, then we know the pagination is working properly.  Let's display 30 items per page.

The data will be created by the following FactoryGirl methods

```ruby
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @posts_list = FactoryGirl.create_list(:post, 31, topic: @topic, user: @user)
    @first_post = @posts_list[0]
    @last_post = @posts_list[-1]
    @last_post_comment = FactoryGirl.create(:comment, post: @last_post, user: @user)
```

@posts_list will naturally be sorted by the date they were created.  Which is why we can set @first_post to the first item in the array, and @last_post to the last item in the array.  Now if we display 30 items per page we should see @last_post, then have to go to the second page to see only @first_post.  Putting this into a test:

```ruby
  scenario "shows 30 posts per page" do
    expect(page).to have_content(@last_post.title)
    expect(page).not_to have_content(@first_post.title)
    click_link "2"
    expect(page).not_to have_content(@last_post.title)
    expect(page).to have_content(@first_post.title)
  end
```

The link to show the post content will display "expand".  When clicked it will use ajax to grab the content of the post and display it in a div with class "content" then change the link to "collapse".  When collapse is clicked this will set the associated div to display:none.  This will change the link back to "expand", however if clicked this shouldn't make another ajax call as the content is just hidden, and just needs to make the content visible again.  [this stackoverflow answer](https://stackoverflow.com/questions/8801845/how-to-make-capybara-check-for-visibility-after-some-js-has-run) shows how to find the elements and test whether they are visible or not with capybara.  However we will use the new expect syntax in the test.  The expand link will be the same route as showing the post so there will be 2 of them with the same link and 30 with the same text.  Therefore we need to find it by explicity find it with data-remote="true" and the link to the post.  Once the post is expanded and the link changes to collapse, there will only be one collapse link on the page and we don't need to explicitly find it.  Putting this all into a test:

```ruby
    expect(page).not_to have_content(@last_post.content)
    expand_link = "a[data-remote='true'][href='/topics/#{@topic.id}/posts/#{@last_post.id}']"
    find(expand_link).click
    expect(page).to have_content(@last_post.content)
    last_post_content = find(:css, ".content")
    click_button "collapse"
    expect(last_post_content).not_to be_visible
    find(expand_link).click
    expect(last_post_content).to be_visible
```

Finally we can do the final test for this feature checking for the topic name, the number of comments for the post (testing for both 0 comments and 1 comment since we have 1 post with a comment) as well as the user information and how long ago it was created.  Putting it all together into a single file:

*spec/features/posts/list_posts_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Listing Posts" do
  before do
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @posts_list = FactoryGirl.create_list(:post, 31, topic: @topic, user: @user)
    @first_post = @posts_list[0]
    @last_post = @posts_list[-1]
    @last_post_comment = FactoryGirl.create(:comment, post: @last_post, user: @user)
    visit "/"
    click_link "Forum"
    click_link @topic.name
  end

  scenario "shows 30 posts per page" do
    expect(page).to have_content(@last_post.title)
    expect(page).not_to have_content(@first_post.title)
    click_link "2"
    expect(page).not_to have_content(@last_post.title)
    expect(page).to have_content(@first_post.title)
  end

  scenario "has a toggle to show content for each post", js: true do
    expect(page).not_to have_content(@last_post.content)
    expand_link = "a[data-remote='true'][href='/topics/#{@topic.id}/posts/#{@last_post.id}']"
    find(expand_link).click
    expect(page).to have_content(@last_post.content)
    last_post_content = find(:css, ".content")
    click_button "collapse"
    expect(last_post_content).not_to be_visible
    find(expand_link).click
    expect(last_post_content).to be_visible
  end

  scenario "shows basic information for each post" do
    expect(page).to have_content(@topic.name)
    expect(page).to have_content("0 comments")
    expect(page).to have_content("1 comment")
    expect(page).to have_content("created less than a minute ago by #{@user.username}")
  end
end
```

We are now greeted with our first error:

```irb
  1) Listing Posts shows 30 posts per page
     Failure/Error: expect(page).not_to have_content(@first_post.title)
```

Right now all of our posts are shown on the index page and it is time to implement the pagination.  Let's add [bootstrap-will_paginate](https://github.com/yrgoldteeth/bootstrap-will_paginate) to the gemfile and run bundle install.  There is another gem called will_paginate-bootstrap but it is no longer maintained.  Then in the topics controller in the show action let's add the paginated @posts instance variable.

*app/controllers/topics_controller.rb*
```ruby
  def show
    @topic = Topic.find(params[:id])
    @posts = @topic.posts.paginate(page: params[:page], per_page: 30)
  end
```

Then in the topics show file replace the topic.posts.each call with @posts.each

*app/views/topics/show.html.erb*
```erb
  <% @posts.each do |post| %>
```

We are now faced with a new error:

```irb
  1) Listing Posts shows 30 posts per page
     Failure/Error: expect(page).to have_content(@last_post.title)
```

Now the page isn't showing the last post.  This means that the page is properly paginating, but the last post is on the second page as the posts are listed in the order they were created.  In order to order them from newest to oldest, we need to use the order method seen [here](https://apidock.com/rails/v4.2.7/ActiveRecord/QueryMethods/order) by applying .order(created_at: :desc) to @topic.posts before we paginate the collection.

*app/controllers/topics_controller.rb*
```ruby
  def show
    @topic = Topic.find(params[:id])
    @posts = @topic.posts.order(created_at: :desc).paginate(page: params[:page], per_page: 30)
  end
```

Now we have the following error:

```irb
  1) Listing Posts shows 30 posts per page
     Failure/Error: click_link "2"

     Capybara::ElementNotFound:
       Unable to find link "2"
```

This link is the link generated by the will_paginate generator which we need to add to the view.  We'll place it on the last line within unless @topics.posts.blank?

*app/views/topics/show.html.erb*
```erb
<%= will_paginate(@posts) %>
```

Now the pagination test passes and we are faced with the following

```irb
     Failure/Error: find(expand_link).click

     Capybara::ElementNotFound:
       Unable to find css "a[data-remote='true'][href='/topics/1/posts/31']"
```

Let's add the div with class post and both the regular and ajax link within the unless post.deleted? conditional.

*app/views/topics/show.html.erb*
```erb
    <% unless post.deleted? %>
      <div class="post">
        <%= link_to post.title, topic_post_path(@topic, post) %>
        <%= link_to "expand", topic_post_path(@topic, post), remote: true, class: "btn btn-default" %>
      </div>
    <% end %>
```

Which gives the following since that link currently doesn't do anything

```irb
  2) Listing Posts has a toggle to show content for each post
     Failure/Error: expect(page).to have_content(@last_post.content)
```

The link is controlled by the show action of the posts controller with a javascript format.  By default this only responds to html, so let's add in a javascript response.

*app/controllers/posts_controller.rb*
```ruby
  def show
    respond_to do |format|
      format.html
      format.js
    end
  end
```

The javascript response invokes the show.js.erb file if it exists.  The first step will be to grab the post div so that we can append the content div to it.  I tried using `this`,  However since this isn't triggered by a javascript listener but by the ruby controller, it just gave me back the window object.  The information that we do have to us is the @post instance variable that the show action has access to.  Therefore just like in the spec, I decided to look for the associated link and grab the parent element which would be the div.  I initially used document.querySelector but that gave plain text with the render since .append needs the jquery object returned with the $ selector.

```javascript
var postDiv = $("a[href='/topics/<%= @post.topic.id %>/posts/<%= @post.id %>']").parent();
```

Now we can make a partial for the post content and add it to the div we found with append.

*app/views/posts/_content.html.erb*
```erb
<div class="content">
  <%= @post.content %>
</div>
```

Putting the file together with the append function

*app/views/posts/show.js.erb*
```javascript
var postDiv = $("a[href='/topics/<%= @post.topic.id %>/posts/<%= @post.id %>']").parent();
postDiv.append("<%=j render 'content' %>");
```

Now we are on to the next error:
```
  1) Listing Posts has a toggle to show content for each post
     Failure/Error: click_button "collapse"

     Capybara::ElementNotFound:
       Unable to find button "collapse"
```

Now we need to delete the expand link as we won't be making another request and replace it will a collapse button.  It would make sense to put the element into its own variable and use it to find the div.  Then we will have it selected to easily replace it with a collapse button.

```javascript
var expandElement = $("a[data-remote='true'][href='/topics/<%= @post.topic.id %>/posts/<%= @post.id %>']");
var postDiv = expandElement.parent();
var collapseElementHTML = '<button class="btn btn-default">collapse</button>';
expandElement.replaceWith(collapseElementHTML);
```

Now we have the following error:

```irb
     Failure/Error: expect(last_post_content).not_to be_visible
       expected `#<Capybara::Node::Element tag="div" path="//HTML[1]/BODY[1]/DIV[1]/DIV[1]/DIV[1]">.visible?` to return false, got true
```

Now that we have a collapse button, we need to add an event listener to toggle the content to be visible or not.  First we need to select the generated button.

```javascript
var collapseElement = postDiv.children('button:first');
```

With it selected, I tried collapseElement.addEventListener("click", function() {}), collapseElement.click(function() {}), and finally collapseElement.on("click", function() {}).  With the first 2 options, the function would be invoked when clicking on the original expand link even though the event is added to the collapseElement.  There would be no function call on the new button.  After some digging, I found out that since I am finding the collapseElement with .children('button:first') it returns a jquery object since I found it with a jquery method.  AddEvenetListener can only be applied to objects found through javascript queries such as document.querySelector or document.getElementBy.

To test if we are working with the correct content in this listener, we will grab the contentDiv by using the siblings method and looking for the first div who is a sibling to our collapseElement then diplay the text in a console.log.

```javascript
collapseElement.on("click", function() {
    var contentDiv = collapseElement.siblings('div:first');
    console.log(contentDiv.text());
});
```

This will add an event listener on the collapse element which was just generated.  If you play around with the generated buttons while looking at the log, you will see that if you make multiple buttons then click on any of them, they will display the text of the last generated content.  This is because the collapseElement variable was made with var collapseElement = postDiv.children('button:first') before the listener function, and therefore it never changes.  We need to find which button was clicked on with $(this).

```javascript
collapseElement.on("click", function() {
  collapseElement = $(this);
  var contentDiv = collapseElement.siblings('div:first');
  console.log(contentDiv.text());
});
```

With some more testing with the console you can see that we are now selecting the appropriate contentDIv.  All that is left is to toggle the div and change the text to collapse or expand appropriately with an if statement.  Putting it all together:

*app/views/posts/show.js.erb*
```javascript
var expandElement = $("a[data-remote='true'][href='/topics/<%= @post.topic.id %>/posts/<%= @post.id %>']");
var postDiv = expandElement.parent();
var collapseElementHTML = '<button class="btn btn-default">collapse</button>';
postDiv.append("<%=j render 'content' %>");
expandElement.replaceWith(collapseElementHTML);

var collapseElement = postDiv.children('button:first');
collapseElement.on("click", function() {
  collapseElement = $(this);
  var contentDiv = collapseElement.siblings('div:first');
  contentDiv.toggle();
  if (collapseElement.text() === "collapse") {
    collapseElement.html("expand");
  } else {
    collapseElement.html("collapse");
  }
});
```

Now we are faced with the following

```irb
  1) Listing Posts has a toggle to show content for each post
     Failure/Error: find(expand_link).click

     Capybara::ElementNotFound:
       Unable to find css "a[data-remote='true'][href='/topics/1/posts/31']"
```

We didn't know exactly what we were going to turn the expand link in to when we wrote the test.  Now we know that we need to search for a button so let's update the test accordingly.

```ruby
find(:css, "button").click
```

Everything passes in that scenario, and now we are on to the easy part of showing basic information

```irb
  1) Listing Posts shows basic information for each post
     Failure/Error: expect(page).to have_content(@topic.name)
```

Let's just put that in an h2 in the show file

```erb
<h2><%= @topic.name %></h2>
```

Then we move onto the next error:

```irb
  3) Listing Posts shows basic information for each post
     Failure/Error: expect(page).to have_content("0 comments")
```

We'll keep the same styling as the topic index page and put an h2 around the post link and a paragraph around the pluralization of comments.

*app/views/topics/show.html.erb*
```erb
<h2><%= link_to post.title, topic_post_path(@topic, post) %></h2>
<p><%= pluralize(post.comments.count, "comment") %></p>
```

Which leaves us with hopefully the last error:

```irb
  1) Listing Posts shows basic information for each post
     Failure/Error: expect(page).to have_content("created less than a minute ago by #{@user.username}")
```

We'll put it in a p tag

```erb
        <p>
          created <%= time_ago_in_words(post.created_at) %> ago by
          <%= link_to "#{post.user.username}", profile_path(post.user.profile) %>
        </p>
```

Putting it all together:

*app/views/topics/show.html.erb*
```erb
<h2><%= @topic.name %></h2>
<% unless @topic.posts.blank? %>
  <% @posts.each do |post| %>
    <% unless post.deleted? %>
      <div class="post">
        <h2><%= link_to post.title, topic_post_path(@topic, post) %></h2>
        <p><%= pluralize(post.comments.count, "comment") %></p>
        <p>
          created <%= time_ago_in_words(post.created_at) %> ago by
          <%= link_to "#{post.user.username}", profile_path(post.user.profile) %>
        </p>
        <%= link_to "expand", topic_post_path(@topic, post), remote: true, class: "btn btn-default" %>
      </div>
    <% end %>
  <% end %>
  <%= will_paginate(@posts) %>
<% else %>
  <p>There are no posts under this topic</p>
<% end %>

<%= link_to "New Post", new_topic_post_path(@topic), class: "btn btn-primary" %>
```

Then we will use the same style as the list of topics by adding it to forum.scss

*app/assets/stylesheets/forum.scss*
```scss
.topic, .post {
	border-bottom: 1px solid #d1d1d1;
	p {
	  color: #888;
	  font-size: small;
	}
}
```

With the posts listing styled, we are going to do a small refactor with adding a post partial [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-7---adding-a-post-partial/)
