---
title: "Creating a Forum in Rails using Behavioral Driven Development - Part 4 - Deleting the Models"
description: "Adding ability to delete records for a forum in Rails"
pubDate: "May 4 2017"
---

Like with editing, we want the ability for a user to delete their post or comment.  However when the user deletes a post, we still want to keep the comments as well as their structure so that other users content is not destroyed.  Likewise when a comment is deleted, we want the child comments to still exist and still be structured in a way that makes sense underneath.

In order to accomplish this, we won't actually be deleting posts or comments.  We will just set their content to the string "[deleted]" and will set a boolean "deleted" to true.  If a post or comment has this variable set to true, it will not show up under a topic or under a users profile.

## Deleting Posts

Pseudo code for deleting a post

- User Logs in
- User clicks forum link
- User clicks on a topic he previously made a post on
- User clicks on the title of the post he previously created
- User sees the button "delete post" as he is the creator and clicks on it
- User sees a confirmation box saying "Are you sure you want to delete your post?" and clicks yes
- User is redirected to the post url where they see [deleted] in place of their title and content
- User goes back to the topic the post was located in and doesn't see the post

Since this test involves javascript with clicking on the alert, we need to add js: true into the scenario.  I also had to read [this stackoverflow answer](http://stackoverflow.com/questions/2458632/how-to-test-a-confirm-dialog-with-cucumber/5002648#comment-8214000) which stated how to aceept a confirmation with page.driver.browser.switch_to.alert.accept.

*spec/features/posts/delete_post_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Deleting a Post" do
  before do
    @owner = FactoryGirl.create(:user)
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic, user: @owner)
    @title = @post.title
    @content = @post.content
  end

  scenario "as the ownder of the post", js: true do
    login_as(@owner)
    visit "/"
    click_link "Forum"
    click_link "#{@topic.name}"
    click_link "#{@post.title}"
    click_link "delete post"
    page.driver.browser.switch_to.alert.accept
    expect(page).to have_content("Your post has been successfully deleted")
    expect(page).not_to have_content(@title)
    expect(page).not_to have_content(@content)
    expect(page).to have_content("[deleted]")
    visit topic_path(@topic)
    expect(page).not_to have_content(@post.title)
  end

  scenario "as another user" do
    login_as(@user)
    visit topic_post_path(@topic, @post)
    expect(page).not_to have_content("delete post")
  end
end
```

We get the first error stating that capybara is missing the selenium-webdriver gem.

```irb
  1) Deleting a Post as the ownder of the post
     Failure/Error: visit "/"

     LoadError:
       Capybara's selenium driver is unable to load `selenium-webdriver`, please install the gem and add `gem 'selenium-webdriver'` to your Gemfile if you are using bundler.
```

Since we are only using this in our tests, you can add it to the :test group in the gemfile and then run bundle install

*Gemfile*
```ruby
group :test do
  gem 'selenium-webdriver'
end
```

```irb
bundle install
```

Now we are getting an error with missing the mozzila geckodriver

```irb
  1) Deleting a Post as the ownder of the post
     Failure/Error: visit "/"

     Selenium::WebDriver::Error::WebDriverError:
        Unable to find Mozilla geckodriver. Please download the server from https://github.com/mozilla/geckodriver/releases and place it somewhere on your PATH. More info at https://developer.mozilla.org/en-US/docs/Mozilla/QA/Marionette/WebDriver.
```

I'm still not too familiar of where to put the actual file, so I found [this stackoverflow answer](http://stackoverflow.com/questions/38100579/how-to-install-and-configure-geckodriver-on-rails-ubuntu) with instructions.

First I downloaded geckodriver-v0.16.1-linux64.tar.gz from https://github.com/mozilla/geckodriver/releases.  Then placed it into the root directory of my ubuntu workspace on cloud 9.  However the file I got is a .tar.gz file and not a .zip file so I found [this answer](https://askubuntu.com/questions/25347/what-command-do-i-need-to-unzip-extract-a-tar-gz-file) for how to unzip it.  Then I ran the following commands

```irb
tar -xvzf geckodriver-v0.16.1-linux64.tar.gz
sudo mv geckodriver /usr/bin/
cd /usr/bin
sudo chmod +x geckodriver
export PATH=$PATH:/usr/bin/geckodriver
```

Runnig the test now i just hangs and when I force exit out of the test it gives the following

```irb
  1) Deleting a Post as the ownder of the post
     Failure/Error: Unable to find matching line from backtrace

     EOFError:
       end of file reached
```

After much more googling I found [this tutorial](http://www.opinionatedprogrammer.com/2011/02/capybara-and-selenium-with-rspec-and-rails-3/#comment-220) which states there is another step needed with the database_cleaner gem.  After adding it to the test group and running bundle install I created the file in the tutorial

*spec/support/database_cleaner.rb*
```ruby
DatabaseCleaner.strategy = :truncation

RSpec.configure do |config|
  config.use_transactional_fixtures = false
  config.before :each do
    DatabaseCleaner.start
  end
  config.after :each do
    DatabaseCleaner.clean
  end
end
```

Then included it inside of rails_helper

*spec/rails_helper.rb*
```ruby
require 'support/database_cleaner'
```

This did not fix anything.  Also when I let the test run for 60 seconds which is apparently the default before it times out I get the following error:

```irb
  1) Deleting a Post as the ownder of the post
     Failure/Error: visit "/"

     Net::ReadTimeout:
       Net::ReadTimeout
```

Trying to access the browser in irb, the process also hangs

```irb
require 'selenium-webdriver'
browser = Selenium::WebDriver.for(:firefox)
```

I'm pretty sure the selenium webdriver needs to be initialized since i set the test to run with javascript.  Then capybara is trying to set the browser in a similar method to the irb lines above and gets hung up there.

After 60 seconds, I'm met with this massive error message

```irb
Net::ReadTimeout: Net::ReadTimeout
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/protocol.rb:158:in `rbuf_fill'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/protocol.rb:136:in `readuntil'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/protocol.rb:146:in `readline'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http/response.rb:40:in `read_status_line'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http/response.rb:29:in `read_new'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1437:in `block in transport_request'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1434:in `catch'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1434:in `transport_request'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1407:in `request'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1400:in `block in request'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:853:in `start'
        from /usr/local/rvm/rubies/ruby-2.3.0/lib/ruby/2.3.0/net/http.rb:1398:in `request'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/http/default.rb:124:in `response_for'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/http/default.rb:78:in `request'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/http/common.rb:63:in `call'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/w3c_bridge.rb:625:in `raw_execute'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/w3c_bridge.rb:109:in `create_session'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/remote/w3c_bridge.rb:69:in `initialize'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/firefox/w3c_bridge.rb:35:in `initialize'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/common/driver.rb:52:in `new'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver/common/driver.rb:52:in `for'
        from /usr/local/rvm/gems/ruby-2.3.0/gems/selenium-webdriver-3.1.0/lib/selenium/webdriver.rb:82:in `for'
        from (irb):3
        from /usr/local/rvm/rubies/ruby-2.3.0/bin/irb:11:in `<main>'2.3.0 :004 >
```

I wasn't able to find a solution here.  However after many hours of searching I found out that selenium requires an external browser to open, and there is something wrong with my setup that is preventing it from doing so.  I  found [this blog post](https://everydayrails.com/2015/01/27/rspec-switch-selenium-poltergeist.html) where I learned about headless javascript testing which doesn't require an external browser to open.

I downloaded the 32bit phantom js from [here](https://github.com/teampoltergeist/poltergeist) then ran the following commands

```irb
tar xvjf phantomjs-1.9.8-linux-i686.tar.bz2
sudo mv phantomjs-1.9.8-linux-i686/bin/phantomjs /usr/bin
cd /usr/bin
sudo chmod +x phantomjs
export PATH=$PATH:/usr/bin/phantomjs
```

Add the poltergiest gem to group: test and run bundle install.

Then add the following to rails_helper.rb

*spec/rails_helper.rb*
```ruby
require 'capybara/poltergeist'
Capybara.javascript_driver = :poltergeist
```

This gave an error saying that it couldn't read the version of javascript, so I went with a reinstall following the advice of [this stackoverflow answer](http://stackoverflow.com/questions/40528895/cliver-and-phantomjs-error-with-update-to-macos-sierra).

Running the following commands to install phantom as a node package, then moving the file into the user bin file then deleting the rest of the node package.

```irb
rm -rf /usr/local/bin/phantomjs
npm -g install phantomjs-prebuilt
sudo mv /home/ubuntu/.nvm/versions/node/v4.1.1/lib/node_modules/phantomjs-prebuilt/lib/phantom/bin/phantomjs /usr/bin/
export PATH=$PATH:/usr/bin/phantomjs
sudo rm -rf /home/ubuntu/.nvm/versions/node/v4.1.1/bin/phantomjs
```

Then we are faced with the following error

```irb
Failure/Error: val = step

     ActiveRecord::StatementInvalid:
       SQLite3::BusyException: database is locked: UPDATE "users" SET "sign_in_count" = ?, "current_sign_in_at" = ?, "last_sign_in_at" = ?, "current_sign_in_ip" = ?, "last_sign_in_ip" = ?, "updated_at" = ? WHERE "users"."id" = ?
```

This database is locked erorr didn't go away with resetting the test database or ActiveRecord::Base.connection.execute("BEGIN TRANSACTION; END;").  Since this error was tied in with signing in I eventually found [this solution in devise documentation](https://github.com/plataformatec/devise/wiki/How-To:-Test-with-Capybara) where I had to include and add another file called share_db_conenction.rb and include it in rails_helper.rb.

*spec/support/share_db_connection.rb*
```ruby
class ActiveRecord::Base
  mattr_accessor :shared_connection
  @@shared_connection = nil

  def self.connection
    @@shared_connection || retrieve_connection
  end
end

# Forces all threads to share the same connection. This works on
# Capybara because it starts the web server in a thread.
ActiveRecord::Base.shared_connection = ActiveRecord::Base.connection
```

Now we get our first error that we can develop with:

```irb
  1) Deleting a Post as the ownder of the post
     Failure/Error: click_link "delete post"

     Capybara::ElementNotFound:
       Unable to find link "delete post"
```

This delete method is going to edit the post (so we are using the put verb) and call a custom method called delete in the post controller.  Let's add the route to config.rb so we know exactly what to call for the button.  This is done using a member do block on the posts route.  We also need to omit the :destroy method.

*config/routes.rb*
```ruby
    resources :posts, except: [:index, :destroy] do
      member do
        put :delete
      end
      resources :comments, except: [:index]
    end
```

then run rake routes and look for the route we just created.  You will find `delete_topic_post PUT  /topics/:topic_id/posts/:id/delete(.:format)                 posts#delete`.  Inside of the post show view inside of the if statement checking if the post user is equal to the current user let's add the button.  Since this isn't a GET request we will have to specify method: :put.

*app/views/posts/_show.html.erb*
```erb
  <%= link_to "delete post", delete_topic_post_path(@post.topic, @post),
                             method: :put, data: { confirm: "Are you sure?" },
                             class: "btn btn-primary btn-xs" %>
```

Which leads us to the following error

```irb
     1.1) Failure/Error: page.driver.browser.switch_to.alert.accept

          NoMethodError:
            undefined method `switch_to' for #<Capybara::Poltergeist::Browser:0x000000061f61b0>
            Did you mean?  switch_to_frame
```

The syntax above is what we were using when the plan was to continue with selenium driver.  Looking around I found that poltergeist doesn't accept confirms/alerts as shown [here](https://github.com/teampoltergeist/poltergeist/issues/50).  I would like to keep this in the test otherwise all of this set up would be for nothing.  In the page someone showed their solution by adding a alert_confirmer file as a support file as shown [here](https://gist.github.com/michaelglass/8610317).

*spec/support/alert_confirmer.rb*
```ruby
module AlertConfirmer
  def reject_confirm_from &block
    handle_js_modal 'confirm', false, &block
  end

  def accept_confirm_from &block
    handle_js_modal 'confirm', true, &block
  end

  def accept_alert_from &block
    handle_js_modal 'alert', true, &block
  end

  def get_alert_text_from &block
    handle_js_modal 'alert', true, true, &block
    get_modal_text 'alert'
  end

  def get_modal_text(name)
    page.evaluate_script "window.#{name}Msg;"
  end

  private

  def handle_js_modal name, return_val, wait_for_call = false, &block
    modal_called = "window.#{name}.called"
    page.execute_script "
    window.original_#{name}_function = window.#{name};
    window.#{name} = function(msg) { window.#{name}Msg = msg; window.#{name}.called = true; return #{!!return_val}; };
    #{modal_called} = false;
    window.#{name}Msg = null;"

    block.call

    if wait_for_call
      timed_out = false
      timeout_after = Time.now + Capybara.default_wait_time
      loop do
        if page.evaluate_script(modal_called).nil?
          raise 'appears that page has changed since this method has been called, please assert on page before calling this'
        end

        break if page.evaluate_script(modal_called) ||
          (timed_out = Time.now > timeout_after)

        sleep 0.001
      end
      raise "#{name} should have been called" if timed_out
    end
  ensure
    page.execute_script "window.#{name} = window.original_#{name}_function"
  end
end
```

Then requiring it in rails_helper and adding it as a feature to the config block

*spec/rails_helper.rb*
```ruby
require 'support/alert_confirmer'
RSpec.configure do |config|
  config.include AlertConfirmer, type: :feature
end
```

Then changing the lines click_link "delete post" and page.driver.browser.switch_to.alert.accept in the delete spec to

*spec/features/posts/delete_post_spec.rb*
```ruby
    accept_confirm_from do
      click_link "delete post"
    end
```

Which gives us the following error.  Just to test the alert_confirmer functionality I did a quick test using reject_confirm_from and it never tried to access the controller which shows that the confirm test is working properly.

```irb
     AbstractController::ActionNotFound:
       The action 'delete' could not be found for PostsController
```

Now we just implement the action where we set the fields to [deleted] and set the deleted boolean to true.  We also need to add :delete to the list of methods :set_post runs before

*app/controllers/posts_controller.rb
```ruby
  before_action :set_post, only: [:edit, :show, :update, :delete]

  def delete
    @topic = @post.topic
    @post.title = "[deleted]"
    @post.content = "[deleted]"
    @post.deleted = true
    @post.save
    flash[:danger] = "Your post has been successfully deleted"
    redirect_to topic_post_path(@topic, @post)
  end
```

We are now faced with the following since we haven't added the boolean to the database

```irb
     NoMethodError:
       undefined method `deleted=' for #<Post:0x000000068f8468>
       Did you mean?  delete
```

We can go ahead and make a migration to add this to both the post model and comment model to save some time.

```irb
rails g migration add_deleted_boolean_to_posts_and_comments
```

Then add the two add_column lines to the resulting migration file

```ruby
class AddDeletedBooleanToPostsAndComments < ActiveRecord::Migration[5.0]
  def change
    add_column :posts, :deleted, :boolean
    add_column :comments, :deleted, :boolean
  end
end
```

Then run the migration and preapre the test database for the new schema.

```irb
rails db:migrate
rails db:test:prepare
```

Surpisingly all tests pass.  However after closer investigation it is because we were testing if @post.title didn't show up under the topic with expect(page).not_to have_content(@post.title).  When we delete the post we aren't changing the @post instance variable in the test.  So change the line to the following.  Also we can remove the @title and @content instance variables in the before do block since the instance variable isn't changing.

*spec/features/delete_post_spec.rb*
```ruby
visit topic_path(@topic)
expect(page).not_to have_content("[deleted]")
```

Then we just need to add a check on the topic page to not display the post if it is deleted

*app/views.topics/show.html.erb*
```erb
  <% @topic.posts.each do |post| %>
    <% unless post.deleted? %>
      <div class="row">
        <%= link_to post.title, topic_post_path(post) %>
      </div>
    <% end %>
  <% end %>
```

Now all tests pass and I want to make a few more additions.  Once a post is deleted, I don't want anyone to have access to edit or delete the post again.  Not that deleting the post again would do anything.  In addition I'll add another check for require_same_user for the delete action

*app/controllers/posts_controller.rb*
```ruby
  before_action :check_deleted, only: [:edit, :update, :delete]

  private
  def check_deleted
    if @post.deleted?
      flash[:danger] = "This post is deleted"
      redirect_to topic_post_path(@post.topic, @post)
    end
  end
```

Then I will also delete the links by adding another conditional onto the same user check

*app/views/posts/_show.html.erb*
```ruby
  <% if @post.user == current_user && !@post.deleted? %>
```

###Comments

Pseudo code for deleting a comment

- User Logs in
- User clicks forum link
- User clicks on a topic
- User clicks on the title of the post he previously commented on
- User clicks the button "delete comment" next to the comment he created
- User sees a confirmation box saying "Are you sure?" and clicks yes
- User is redirected to the comment url where they see [deleted] in place of their content

*spec/features/comments/delete_comment_spec.rb*
```ruby
require 'rails_helper'

RSpec.feature "Deleting a Comment" do
  before do
    @owner = FactoryGirl.create(:user)
    @user = FactoryGirl.create(:user)
    @topic = FactoryGirl.create(:topic)
    @post = FactoryGirl.create(:post, topic: @topic)
    @comment = FactoryGirl.create(:comment, post: @post, user: @owner)
  end

  scenario "as the owner of the post", js: true do
    login_as(@owner)
    visit "/"
    click_link "Forum"
    click_link "#{@topic.name}"
    click_link "#{@post.title}"
    accept_confirm_from do
      click_link "delete comment"
    end
    expect(page).to have_content("Your comment has been successfully deleted")
    expect(page).not_to have_content(@comment.content)
    expect(page).to have_content("[deleted]")
  end

  scenario "as another user" do
    login_as(@user)
    visit topic_post_comment_path(@topic, @post, @comment)
    expect(page).not_to have_content("delete comment")
  end
end
```

This gives the following as expected

```irb
  1) Deleting a Comment as the owner of the post
     Failure/Error: click_link "delete comment"

     Capybara::ElementNotFound:
       Unable to find link "delete comment"
```

Let's add the button to the comment show partial within the check statement for the current user.  The path for deleting a comment will be the same format as for deleting a post.  We can also add the statement to check if the comment is deleted within the conditional

*app/views/comments/_show.html.erb*
```erb
  <% if comment.user == current_user && !comment.deleted %>
    <%= link_to "delete comment", delete_topic_post_comment_path(@post.topic, @post, comment),
                                  method: :put, data: { confirm: "Are you sure?" },
                                  class: "btn btn-primary btn-xs" %>
  <% end %>
```

Which gives us an undefined route as we haven't added it yet

```irb
     ActionView::Template::Error:
       undefined method `delete_topic_post_comment_path' for #<#<Class:0x00000009b941d8>:0x00000009d04748>
```

Let's add the delete put route and remove the destroy action

*config/routes.rb*
```ruby
      resources :comments, except: [:index, :destroy] do
        member do
          put :delete
        end
      end
```

Which gives us an action nor found error

```irb
     AbstractController::ActionNotFound:
       The action 'delete' could not be found for CommentsController
```

Let's delete the comment with the same logic as deleting a post.  We also need to add the delete action to the set_comment and require_same_user methods.  Let's also add the check_deleted function while we are here

*app/controllers/comments_controller.rb*
```ruby
  before_action :set_comment, only: [:edit, :show, :update, :delete]
  before_action :require_same_user, only: [:edit, :update, :delete]
  before_action :check_deleted, only: [:edit, :update, :delete]

  def delete
    @post = @comment.post
    @topic = @post.topic
    @comment.content = "[deleted]"
    @comment.deleted = true
    @comment.save
    flash[:danger] = "Your comment has been successfully deleted"
    redirect_to topic_post_comment_path(@topic, @post, @comment)
  end

  private

  def check_deleted
    if @comment.deleted?
      flash[:danger] = "This comment is deleted"
      redirect_to topic_post_comment_path(@comment.post.topic, @comment.post, @comment)
    end
  end
```

Now all specs pass and we successfully added all CRUD functionality needed for this forum.  Now we just need to add stying and add more information to the posts such as time created, a link to the user who created the comment or post.  Then eventually add a link to all the users posts and comments within their profile.  The next post can be found [here](/blog/creating-a-forum-in-rails-using-behavioral-driven-development---part-5---styling-and-adding-information-to-topics/)
