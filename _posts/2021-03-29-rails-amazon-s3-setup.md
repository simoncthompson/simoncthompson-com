---
layout: post
title:  "Setting up Rails and AWS S3 for file storage and user uploads"
date:   2021-03-29 14:00:00 +0800
categories: saas
permalink: /rails-amazon-s3-setup/
---

I recently found myself needing a fairly simple S3 setup so that users (and myself) could upload files to my Rails application.

Most of the resources I found were either overly complicated, or skipped over crucial elements, or both.

So I decided to write a simple guide to getting S3 set up in your Rails app. I hope you find it helpful!


Setup the Amazon S3 bucket, user, and policy
--------------------------------------------

The first thing you'll need to do is [create an AWS account](https://aws.amazon.com/), if you haven't done so already. This is fairly straightforward so I won't cover it in this post. Once you've done that, we can start to create your S3 bucket.

### Create a new bucket in AWS

Select S3 as your service in the search bar.

![AWS search](/images/post-rails-amazon-s3-setup/aws-search.png)

Click "Create bucket"

![AWS create bucket](/images/post-rails-amazon-s3-setup/aws-create-bucket.png)

Give your bucket a name, and select a region. Note that the region you choose will have a slight effect on speed - the closer your user is to the bucket region, the faster the requests will be. So choose the region where you will have most users.

![AWS create bucket form](/images/post-rails-amazon-s3-setup/aws-create-bucket-form.png)

You can leave the rest of the settings as their defaults, including 'Block all public access' settings.

That is unless you will need to expose the links to the files to your users. For example, I run a podcast hosting service (Incast.fm). With podcasts, I need to make links to audio and images publicly available on RSS feeds, so that Apple Podcasts, Spotify etc. can pick them up and add them to their apps. For that use case I need to do some tweaking with the public settings, which I've detailed how to do at the end of this post.

But if you just need to give users the ability to upload files and the only place they will need to see them is in your app (e.g. user avatars etc.), then it is ok to block public access.

Ok - so we have our bucket, now we need to add our app as a "user", and assign a policy. We'll actually start with the policy creation, and then assign that to a user.

### Create a new IAM policy

![AWS IAM search](/images/post-rails-amazon-s3-setup/aws-iam-search.png)

Navigate to 'Policies' on the side panel, and click 'Create policy'.

![AWS create policy](/images/post-rails-amazon-s3-setup/aws-create-policy.png)

In the "Create policy" section, you'll need to select a few options:

In the "Service" section, select S3. Simple.

In the "Actions" section, you'll need to select 4 actions allowed in S3.

-   ListBucket

-   GetObject

-   PutObject

-   DeleteObject

![AWS policy options](/images/post-rails-amazon-s3-setup/aws-policy-options.png)

In the resources section, you want to select "Specific" resources for each action.

For "Bucket" actions, you want to select "Add ARN"

![AWS policy options 2](/images/post-rails-amazon-s3-setup/aws-policy-options-2.png)

And then type in the bucket name you just created, then click "Add".

![AWS policy ARN](/images/post-rails-amazon-s3-setup/aws-policy-arn.png)

For the "Object" actions, you can just select "Any".

![AWS policy ARN 2](/images/post-rails-amazon-s3-setup/aws-policy-arn-2.png)

Click to the next page to add tags if you want, but you can just leave this blank (as I have).

From the "Tags" page, click next to review the policy. All you need to do is add a name for the policy, and you're set! 

![AWS policy review](/images/post-rails-amazon-s3-setup/aws-policy-review.png)

Ok - now we have the policy, and we need to create a user to attach that policy to.

### Add a new IAM user

Go back to the IAM panel and select "Users" from the side panel, and "Add user".

![AWS IAM create user](/images/post-rails-amazon-s3-setup/aws-iam-create-user.png)

This "user" is actually your application, so you want to give them a name, and then select "Programmatic access". This will allow you to get the keys to enter in your application and access S3.

![AWS add user](/images/post-rails-amazon-s3-setup/aws-add-user.png)

Next, we want to attach our policy that we just created to this user. Select "Attach existing policies directly", and select your newly created policy.

![AWS attach policy to user](/images/post-rails-amazon-s3-setup/aws-user-attach-policy.png)

You can click next and add tags if you want, but again, you don't need to.

On the review page, you can click "Create user", which will take you through to the page to get your keys.

Note: you won't be able to access these keys again after you leave this page, so make sure you save them. You can do that by downloading them as a CSV, or simply writing them down (or adding them to your app).

![AWS download user credentials](/images/post-rails-amazon-s3-setup/aws-user-credentials.png)

And that's it as far as AWS is concerned, now we can move onto our application :)

Install ActiveStorage
---------------------

If you're not aware already, Active Storage is the Rails engine that facilitates uploading files to a cloud storage service (like AWS S3). It used to be a gem, but now is just included in Rails 5.2 and above - we just need to install it.

In your terminal, write the following commands.

```ruby
rails active_storage:install # this will create a migration, among other things
rails db:migrate # run the migration
```

Note: this will work only in Rails 5.2 and above. In earlier versions ActiveStorage is actually a gem you'll need to add (which I won't cover in this post).

Add these gems to your gemfile:

```ruby
gem 'dotenv-rails', groups: [:development, :test] # we'll need a .env file to add your S3 keys
gem 'aws-sdk-s3', require: false
```

And run bundle install.

We'll now need to create your `.env` file:

`touch .env`

And it's very important we add the file to your .gitignore file, as we don't want anyone to be able to see those keys in your repository. You can add this line of code to your .gitignore file:

`*.env`

Then add your keys in the .env file.

```ruby
AWS_ACCESS_KEY_ID=[your access key]
AWS_SECRET_ACCESS_KEY=[your secret key]
S3_BUCKET=[your bucket name]
```

Note: since we're not adding our .env file to our remote repository, our production environment won't have access to it. That means we will also need to add these environment variables to our production server whatever that may be (i.e. Heroku, Digital Ocean etc.). I won't cover this post but it's fairly straightforward in most cloud services.

Now we need to set your Rails app's Active Storage service to use Amazon/AWS.

In your `config/environments/development.rb` AND in `config/environments/production.rb`, add this line of code:

```ruby
config.active_storage.service = :amazon
```

We'll need to create an aws initialiser file to use S3 in your app:


`touch config/initializers aws.rb`

And add this code to it:

```ruby
require 'aws-sdk-s3'

Aws.config.update({
  region: # [your bucket's region e.g. 'ap-southeast-2'],
  credentials: Aws::Credentials.new(ENV['AWS_ACCESS_KEY_ID'], ENV['AWS_SECRET_ACCESS_KEY']),
})
S3_BUCKET = Aws::S3::Resource.new.bucket(ENV['S3_BUCKET'])
```

We need to enter your credentials for the S3 bucket and user we just created. In config/storage.yml, add this code:

```ruby
amazon:
  service: S3
  access_key_id: ENV['AWS_ACCESS_KEY_ID'] 
  secret_access_key: ENV['AWS_SECRET_ACCESS_KEY']
  region: # [your bucket's region e.g. 'ap-southeast-2']
  bucket: # [write the bucket name in full here - don't use an ENV variable]

```

Note: in this `storage.yml` file, for some reason, you are not able to use your `ENV['S3_BUCKET']` variable - you'll need to write the bucket name directly into this file.

Note: there are other ways to add credentials, but using a `.env` file is the simplest in my opinion, and quite secure.

And then the last step is to update your model to allow it to accept files as attachments.

Simply go the model in question, and add the following line to it:

```ruby
has_one_attached :image
```

`:image` can be anything - it's just a placeholder we'll use in our forms. If your working with audio, you could include `:audio` or `:podcast` - you can set the name to whatever you want, like a variable.

You can also use `has_many_attached` if you want to attach multiple files to the instance of the model.

And that's it! Now you can add 'file' fields to forms and your users can start to upload away :)

```ruby
<%= simple_form_for [@podcast, @episode], data: { turbo: false } do |f| %>
  <%= f.input :image, as: :file %>
  <%= f.input :title %>
  <%= f.submit "Save" %>
<% end %>
```

As long as you've included these inputs as permitted parameters in your forms, they will attach themselves to the model whenever a user fills out the form.

And to display these images (or S3 files of any kind), you can call `.image` or `.audio` (or whatever you named your attachment in the model) at any point to access the attachment. For example:

`<%= image_tag @episode.image %>`

And there you have it. Simple S3 set up for Rails.

[Optional] Making your S3 links publicly accessible outside of your Rails app
-----------------------------------------------------------------------------

I mentioned that sometimes you may want to make your S3 links publicly accessible. For example, in my podcast hosting platform ([Incast.fm](http://incast.fm)), I need to expose S3 links in the RSS feeds we generate for users.

To do this we just need to change a few settings in the "Permissions" tab of our bucket.

So. Navigate to your bucket, and select the "Permissions" tab. In the "Block public access" section, choose to edit this section, and uncheck the "Block all public access" checkbox.

![AWS public access](/images/post-rails-amazon-s3-setup/aws-public-access.png)

Further down the Permissions page, you'll see a "Bucket Policy" section. You'll need to add this code:

![AWS public policy JSON](/images/post-rails-amazon-s3-setup/aws-json-policy.png)

```javascript
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "AllowPublicRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::incast-podcasts/*"
        }
    ]
}
```

Lastly, scroll down to "Access control list (ACL)". In the "Everyone (public access)" row, the Bucket ACL value should say "Read". This means that anyone who has one of your S3 links can access the file.

![AWS public read](/images/post-rails-amazon-s3-setup/aws-public-read.png)

If it does not say "Read" here, click Edit, and update this so that anyone with a link can access the file.

Note that only "you" (the application) should have access to write to the bucket (which will include your application's users).

And that's it! That is how you can allow anyone to publicly access your AWS S3 files, provided they have the link.

I hope you found this post helpful, please send me an email if you have any feedback on it.

Cheers, 

Simon

simonct.sct@gmail.com
