# Sinatra Heroku Demo App

You'll need to have the [Heroku Toolbelt](https://toolbelt.heroku.com/) installed for this to work.

- [Getting Started](#getting-started)
- [Getting Heroku configured with IDE](#Heroku-and-the-learn-ide)
- [Configuring Your Database](#configuring-your-database)

# Getting Started

We'll be using the Corneal app generator to start off our sinatra app. To use the gem, let's first install it:

```gem install corneal```

After the gem is installed, we can bootstrap a new Sinatra app by using the `corneal` new command and passing the name of our app:

```corneal new sinatra-heroku```

After our new app is created, let's cd into the directory and run `bundle install` to create a Gemfile.lock. After that, let's initialize a new git repository there, add all the files and make an initial commit. 

Now, the instructions are a bit different for those who are working locally. If you are still using the Learn IDE, then you can skip to [that section]((#Heroku-and-the-learn-ide)) below, otherwise continue here and skip that section when you come to it.

# Setting up Heroku Toolbelt (local env only)
Right away, let's set up a heroku remote that we can push to. To do this, let's install the [Heroku Toolbelt](https://devcenter.heroku.com/articles/heroku-cli) and runn `heroku login` to connect it to our account. After we've done, that we'll be able to run `heroku apps:create` from our terminal.

`heroku apps:create`

we'll see something like this:

```
Creating app... done, ⬢ guarded-journey-60283
https://guarded-journey-60283.herokuapp.com/ | https://git.heroku.com/guarded-journey-60283.git
```
And, when you run `git remote -v` you'll see something like this:
```
heroku  https://git.heroku.com/guarded-journey-60283.git (fetch)
heroku  https://git.heroku.com/guarded-journey-60283.git (push)
```
then we can run `git push heroku master` to deploy the app.  

# Heroku and the Learn IDE

- Create a new SSH key so that the IDE can talk to Heroku
- Create a new app through the heroku dashboard
- Add the heroku git repo as a remote to your local repo

### Add an SSH Key
First, we need to create a new SSH key so that the IDE and Heroku can communicate securely. To do this, run

```
ssh-keygen -t rsa -b 4096 -C "yourgithub@email.com"
```
hit enter to accept the default location and no passphrase. After that, run:
``` 
cat ~/.ssh/id_rsa.pub > key.txt
```
then open key.txt in the text editor. Copy all of the content and go into your [Heroku dashboard's Account Settings page](https://dashboard.heroku.com/account) to add your key to your account.

### Create a new app

Go to the [apps page](https://dashboard.heroku.com/apps) in your dashboard and click the new button on the top right, select create new app.

Give your app a name, for this one, I chose sinatra-heroku-demo. You'll want to copy the name you've given the app for the next step.

### Add a remote pointing to your Heroku git repo

Once you've got the app created, you'll want to create a remote pointing to the heroku git repo that you'll use for deployment.

```
git remote add heroku git@heroku.com:sinatra-heroku-demo.git
```

After this step is done, we should be able to run 
```
git push heroku master
``` 

to deploy our app.

# Configuring Your Database

When I did this for the first time, the build was rejected because sqlite3 is not supported on Heroku. So, to make sure we can get our app deployed, we'll want to configure the app to use a postgresql database in production. For now, we'll keep our development database as sqlite. To set this up, let's create a config/database.yml file:

```
development:
  adapter: sqlite3
  encoding: unicode
  database: db/development.sqlite3
  pool: 5
production:
  adapter: postgresql
  encoding: unicode
  pool: 5
  host: <%= ENV['DATABASE_HOST'] %>
  database: <%= ENV['DATABASE_NAME'] %>
  username: <%= ENV['DATABASE_USER'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

and we'll need to update our config/environment.rb file to use this file to establish a connection to our DB. To do that, replace these lines
```
ActiveRecord::Base.establish_connection(
  :adapter => "sqlite3",
  :database => "db/#{ENV['SINATRA_ENV']}.sqlite"
)
```
with 
```
set :database_file, "./database.yml"
```

Finally, we need to create a development group in our Gemfile so we can still use sqlite3 locally, while allowing heroku to use postgres. It should look like this when you're through:

```
source 'http://rubygems.org'

gem 'sinatra'
gem 'activerecord', '~> 4.2', '>= 4.2.6', :require => 'active_record'
gem 'sinatra-activerecord', :require => 'sinatra/activerecord'
gem 'pg', '0.20'
gem 'rake'
gem 'require_all'
gem 'thin'
gem 'bcrypt'

group :development do
  gem 'sqlite3'
  gem 'shotgun'
  gem 'tux'
  gem 'pry'
end

group :test do
  gem 'rspec'
  gem 'capybara'
  gem 'rack-test'
  gem 'database_cleaner', git: 'https://github.com/bmabey/database_cleaner.git'
end

```

## README NOT COMPLETE

The deploy may work, but when we try to run `heroku open` to view the site in the browser, we'll most likely see an error. When we visit https://dashboard.heroku.com/apps/guarded-journey-60283/logs we'll be able to see the logs and check out the error. But, the default errors here are not helpful. To get more helpful errors in the logs, let's make a few changes. 

1. Add the rails_12factor gem to your Gemfile.
```
gem 'rails_12factor'
```
2. Add the foreman gem to your Gemfile
```
gem 'foreman'
```
3. Create a Procfile in the root of your project and add the following content:
```
web: bundle exec rackup -p $PORT
release: bundle exec rake db:migrate
```

When I did this for the first time, I was able to see an error about my migrations being pending. That's how I googled to find the release option in the Procfile above that handles making sure heroku runs my migrations after a push.

You'll also want to make sure you add in a SESSION_SECRET environment variable. To do this, navigate to the settings page for your app on the heroku dashboard. For me, this is at: https://dashboard.heroku.com/apps/guarded-journey-60283/settings. Then click the button that says Reveal Config Vars. Add in another one called SESSION_SECRET to ensure that your sessions remain secure. To generate the secret, install the session_secret_generator gem and then run `generate_secret`

```
gem install session_secret_generator
generate_secret
```

Copy the secret from your terminal output and add it both in the SESSION_SECRET config var on heroku, and in your .env file locally. It should look like this in your .env file:

```
SESSION_SECRET=8ad90be1a5a9aaaf04a0a99d8efb42c825f16b8fef603f65b600c91d66a17bdd520099130ed70669409a524a97c8f62e9434a0ad102624f9bcff0832e3c2f568
```

**NOTE** Don't use this secret!!! This secret should be kept private and out of version control.