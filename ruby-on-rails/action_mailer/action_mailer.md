# How to send an e-mail?

Emails can be send with Action Mailer. Let's start a fresh project with a minimal User scaffold:

```bash
rails new shop
cd shop
rails g scaffold User first_name last_name email password:digest
rake db:migrate
```

Don't forget to activate the `bcrypt` gem in your Gemfile and run `bundle` after that change. Otherwise you can't use `has_secure_password` and Rails will display an error page.

The user model has a minimal validation for the e-mail address:

**app/models/user.rb**

```ruby
class User < ActiveRecord::Base
  has_secure_password

  validates :email,
            presence: true,
            format: { with:
            /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
            }
end
```

**Task:** We want to send a welcome email to every new user.

## Create the Mailer

The mailer generator script does the hard work for us:

```bash
$ rails g mailer UserMailer
      create  app/mailers/user_mailer.rb
      create  app/mailers/application_mailer.rb
      invoke  erb
      create    app/views/user_mailer
      create    app/views/layouts/mailer.text.erb
      create    app/views/layouts/mailer.html.erb
      invoke  test_unit
      create    test/mailers/user_mailer_test.rb
      create    test/mailers/previews/user_mailer_preview.rb
$
```

The new directory `app/mailers/` uses the same logic as `app/controllers` directory. It contains one `application_mailer.rb` file which contains the following defaults:

**app/mailers/application_mailer.rb**

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: "from@example.com"
  layout 'mailer'
end
```

To send a welcome email to new users we create a new `welcome_email` method in the file `app/mailers/user_mailer.rb`.

**app/mailers/user_mailer.rb**

```ruby
class UserMailer < ApplicationMailer
  default from: 'service@example.com'

  def welcome_email(user)
    @user = user
    mail(to: @user.email, subject: 'Welcome to the Example Shop')
  end
end
```

Action mailer work like normal Rails websites. After creating the method in `app/mailers/user_mailer.rb` you need to create a view in the directory `app/views/user_mailer/`. We start with a standard text email which has the file extension `text.erb`.

**app/views/user_mailer/welcome_email.text.erb**

```text
Hi <%= @user.first_name %>!

we just created a new account for the e-mail
address <%= @user.email %>.

Thanks for joining and have a great day!

- A Script
```

Now we can use the rails console to create a new user:

```bash
$ rails c
Loading development environment (Rails 4.2.0)
>> User.create(first_name: 'Peter', last_name: 'Smith', email: 'peter@example.com', password: '1234', password_confirmation: '1234')
   (0.1ms)  begin transaction
  SQL (0.4ms)  INSERT INTO "users" ("first_name", "last_name", "email", "password_digest", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?, ?)  [["first_name", "Peter"], ["last_name", "Smith"], ["email", "peter@example.com"], ["password_digest", "$2a$10$ZUxY0IyxalbmzR/pcmMHWe6l1EC7a6sCuH08w38cQ2boo9OB3n9J2"], ["created_at", "2015-01-14 13:16:01.042239"], ["updated_at", "2015-01-14 13:16:01.042239"]]
   (1.6ms)  commit transaction
=> #<User id: 1, first_name: "Peter", last_name: "Smith", email: "peter@example.com", password_digest: "$2a$10$ZUxY0IyxalbmzR/pcmMHWe6l1EC7a6sCuH08w38cQ2b...", created_at: "2015-01-14 13:16:01", updated_at: "2015-01-14 13:16:01">
>>
```

and to send him an e-mail:

```bash
>> UserMailer.welcome_email(User.first).deliver_later
  User Load (0.2ms)  SELECT  "users".* FROM "users"  ORDER BY "users"."id" ASC LIMIT 1
Enqueued ActionMailer::DeliveryJob (Job ID: 5fce9b75-f32c-407d-ad51-495e63b689bb) to Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/1
  User Load (0.2ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 1]]
Performing ActionMailer::DeliveryJob from Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/1
  Rendered user_mailer/welcome_email.text.erb within layouts/mailer (1.7ms)

UserMailer#welcome_email: processed outbound mail in 193.7ms

Sent mail to peter@example.com (6.2ms)
Date: Wed, 14 Jan 2015 14:26:07 +0100
From: service@example.com
To: peter@example.com
Message-ID: <54b66e6fdad05_40f83fdc68865be4412f5@Millennium-Falcon.local.mail>
Subject: Welcome to the Example Shop
Mime-Version: 1.0
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

Hi Peter!

we just created a new account for the e-mail
address peter@example.com.

Thanks for joining and have a great day!

- A Script


Performed ActionMailer::DeliveryJob from Inline(mailers) in 201.64ms
=> #<ActionMailer::DeliveryJob:0x007fb8d8bf0228 @arguments=["UserMailer", "welcome_email", "deliver_now", #<User id: 1, first_name: "Peter", last_name: "Smith", email: "peter@example.com", password_digest: "$2a$10$ZUxY0IyxalbmzR/pcmMHWe6l1EC7a6sCuH08w38cQ2b...", created_at: "2015-01-14 13:16:01", updated_at: "2015-01-14 13:16:01">], @job_id="5fce9b75-f32c-407d-ad51-495e63b689bb", @queue_name="mailers">
>> exit
$
```

Of course in the development environment no emails are actually send. You can use the log file or the console to see what email is send.

An email can be send now with `deliver_now` or queued with Active Job with `deliver_later`. The default behaviour for Active Job (Rails' queuing system) is to not queue at all but to do everything right away. So you can use `deliver_later` and switch to queueing later without touching the ActionMailer code again.

To send a new user automatically a welcome email we need to add a callback in the User model:

**app/models/user.rb**

```ruby
class User < ActiveRecord::Base
  has_secure_password

  validates :email,
            presence: true,
            format: { with:
            /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
            }

  after_create :send_welcome_email

  private

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
```

Now a welcome email gets send when ever a new user is created:

```bash
$ rails c
Loading development environment (Rails 4.2.0)
>> User.create(first_name: 'John', last_name: 'Smith', email: 'john@example.com', password: '1234', password_confirmation: '1234')
   (0.1ms)  begin transaction
  SQL (0.3ms)  INSERT INTO "users" ("first_name", "last_name", "email", "password_digest", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?, ?)  [["first_name", "John"], ["last_name", "Smith"], ["email", "john@example.com"], ["password_digest", "$2a$10$x.wWdKH8pySIIAwPa2OX7O/DwZJB5X.sAJJxv3XNvE5bYOTb3GfAO"], ["created_at", "2015-01-14 13:24:33.334473"], ["updated_at", "2015-01-14 13:24:33.334473"]]
Enqueued ActionMailer::DeliveryJob (Job ID: dcff52a8-73d1-4b05-832f-ae673485aeb7) to Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/2
  User Load (0.2ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 2]]
Performing ActionMailer::DeliveryJob from Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/2
  Rendered user_mailer/welcome_email.text.erb within layouts/mailer (1.1ms)

UserMailer#welcome_email: processed outbound mail in 179.4ms

Sent mail to john@example.com (9.8ms)
Date: Wed, 14 Jan 2015 14:24:33 +0100
From: service@example.com
To: john@example.com
Message-ID: <54b66e11890dd_40f53fdc68865be441293@Millennium-Falcon.local.mail>
Subject: Welcome to the Example Shop
Mime-Version: 1.0
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

Hi John!

we just created a new account for the e-mail
address john@example.com.

Thanks for joining and have a great day!

- A Script


Performed ActionMailer::DeliveryJob from Inline(mailers) in 190.9ms
   (0.6ms)  commit transaction
=> #<User id: 2, first_name: "John", last_name: "Smith", email: "john@example.com", password_digest: "$2a$10$x.wWdKH8pySIIAwPa2OX7O/DwZJB5X.sAJJxv3XNvE5...", created_at: "2015-01-14 13:24:33", updated_at: "2015-01-14 13:24:33">
>> exit
$
```

## How to send HTML emails?

If you want to send an HTML email you have to create a `welcome_email` with an `html.erb` file extension and write the HTML content like you would in a normal view:

**app/views/user_mailer/welcome_email.html.erb**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Hi <%= @user.first_name %>!</h1>

    <p>we just created a new account for the e-mail
    address <%= @user.email %>.</p>

    <p>Thanks for joining and have a great day!</p>

    <p>- A Script</p>
  </body>
</html>
```

In case you have an `html.erb` and a `text.erb` file ActionMailer will detect it and will send a `multipart/alternative` email.

Let's create a new user to test this:

```bash
$ rails c
Loading development environment (Rails 4.2.0)
>> User.create(first_name: 'Robert', last_name: 'Smith', email: 'robert@example.com', password: '1234', password_confirmation: '1234')
   (0.1ms)  begin transaction
  SQL (0.4ms)  INSERT INTO "users" ("first_name", "last_name", "email", "password_digest", "created_at", "updated_at") VALUES (?, ?, ?, ?, ?, ?)  [["first_name", "Robert"], ["last_name", "Smith"], ["email", "robert@example.com"], ["password_digest", "$2a$10$Zeh4iwu7LDxiV5J197ES.exiIV88euHzQenJz00k9vfd2iGWA.hb6"], ["created_at", "2015-01-14 13:39:32.034317"], ["updated_at", "2015-01-14 13:39:32.034317"]]
Enqueued ActionMailer::DeliveryJob (Job ID: 79546742-4efc-41bd-ad06-6d20b8df5a88) to Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/3
  User Load (0.2ms)  SELECT  "users".* FROM "users" WHERE "users"."id" = ? LIMIT 1  [["id", 3]]
Performing ActionMailer::DeliveryJob from Inline(mailers) with arguments: "UserMailer", "welcome_email", "deliver_now", gid://shop/User/3
  Rendered user_mailer/welcome_email.html.erb within layouts/mailer (2.3ms)
  Rendered user_mailer/welcome_email.text.erb within layouts/mailer (0.4ms)

UserMailer#welcome_email: processed outbound mail in 109.0ms

Sent mail to robert@example.com (7.0ms)
Date: Wed, 14 Jan 2015 14:39:32 +0100
From: service@example.com
To: robert@example.com
Message-ID: <54b6719440202_41123fdc68865be44138@Millennium-Falcon.local.mail>
Subject: Welcome to the Example Shop
Mime-Version: 1.0
Content-Type: multipart/alternative;
 boundary="--==_mimepart_54b671943ed5f_41123fdc68865be441295";
 charset=UTF-8
Content-Transfer-Encoding: 7bit


----==_mimepart_54b671943ed5f_41123fdc68865be441295
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

Hi Robert!

we just created a new account for the e-mail
address robert@example.com.

Thanks for joining and have a great day!

- A Script


----==_mimepart_54b671943ed5f_41123fdc68865be441295
Content-Type: text/html;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

<html>
  <body>
    <!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Hi Robert!</h1>

    <p>we just created a new account for the e-mail
    address robert@example.com.</p>

    <p>Thanks for joining and have a great day!</p>

    <p>- A Script</p>
  </body>
</html>

  </body>
</html>

----==_mimepart_54b671943ed5f_41123fdc68865be441295--

Performed ActionMailer::DeliveryJob from Inline(mailers) in 117.61ms
   (0.6ms)  commit transaction
=> #<User id: 3, first_name: "Robert", last_name: "Smith", email: "robert@example.com", password_digest: "$2a$10$Zeh4iwu7LDxiV5J197ES.exiIV88euHzQenJz00k9vf...", created_at: "2015-01-14 13:39:32", updated_at: "2015-01-14 13:39:32">
>> exit
$
```

## How to add an Attachment?

If you want to attach a the file `welcome.pdf` to an email you can do this with the `attachments[]` method.

**app/mailers/user_mailer.rb**

```ruby
class UserMailer < ApplicationMailer
  default from: 'service@example.com'

  def welcome_email(user)
    @user = user
    attachments['abc.pdf'] = File.read(Rails.root.join('public', 'abc.pdf'))
    mail(to: @user.email, subject: 'Welcome to the Example Shop')
  end
end
```

### How to add inline Attachments?

To use an inlines attachment in an HTML email you need to use `attachments.inline[]` in the ApplicationMailer and change the html template accordingly.

Here is an example for including a `logo.png` file:

**app/mailers/user_mailer.rb**

```ruby
class UserMailer < ApplicationMailer
  default from: 'service@example.com'

  def welcome_email(user)
    @user = user
    attachments['logo.png'] = File.read(Rails.root.join('app', 'assets', 'images', 'logo.png'))
    mail(to: @user.email, subject: 'Welcome to the Example Shop')
  end
end
```

**app/views/user_mailer/welcome_email.html.erb**

```html
<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <%= image_tag attachments['logo.png'].url %>

    <h1>Hi <%= @user.first_name %>!</h1>

    <p>we just created a new account for the e-mail
    address <%= @user.email %>.</p>

    <p>Thanks for joining and have a great day!</p>

    <p>- A Script</p>
  </body>
</html>
```

No you can send those fancy HTML news letters. ;-)

## More Information

Have a look at http://guides.rubyonrails.org/action_mailer_basics.html to get a complete overview of most ActionMailer features.
