---
published: false
---
## Ruby on Rails - Authenticate Danish NemID with Devise and Auth0

### Introduction

A project that I worked on required the option to let users sign in with their "e-ID", the Danish NemID login service.
NemID used for signing in to online banking, social customer service and other stuff that require validation of the users identity.

I used alot of time on researching different possibilities, on how to implement NemID in Ruby on Rails - and there isn't really any easy plug-and-play solution, yet. A solution was to use JRuby along with NemID's java-applet. Another solution was a custom made Devise integration.

But then I stumpled upon Criipto and their integrations in Auth0.

[Criipto](https://criipto.com/) (former Grean) is a supplier of a service that lets you use NemID together with [Auth0](https://auth0.com/) - and Auth0 is used for SSO(Single Sign On), and can be integrated in many languages. Criipto also has the options for use of Norwegian and Swedish login services.

Auth0 made a very fine starting point [here](https://auth0.com/authenticate/rails/nemid/) and [here](https://auth0.com/docs/quickstart/webapp/rails) on how to get up and running with their and Criipto's service.

But it wasn't really enough for our quest, to get it working in a Rails app with Devise aswell. So I will try to outline that in the following steps, with what I did to get everything working.




### Prerequisites

Make sure you have a working Rails app, atleast at Rails 4.2.10.

The tutorial have been tested with Rails 4.2.10 and Rails 5.1.4, Ruby 2.4.3 on RVM.

### 1. Register with Criipto and Auth0

Go to https://auth0.com/authenticate/rails/nemid/ and follow the guide.

#### Configure Auth0 connection

Log in to your Auth0 account

Click on "Connections" -> "Enterprise", you should see the same as below:
![auth0]({{site.baseurl}}/_posts/auth0.PNG)

Click on "ADFS" or the icon with the bullet lines
![ADFS]({{site.baseurl}}/_posts/adfs.PNG)

You should now see this popup:
![ADFS]({{site.baseurl}}/_posts/adfs2.PNG)

Click on the "cog" icon for "easyid-adfs-DK-NemID-POCES" to access settings for this tenant. (The name might differ from what you have chosen).

In here you can provide the email domains that should be allowed to use for the login services. I just used "@gmail.com" for now.

Hit "Save", and close the dialog.

#### Configure Auth0 client

If you have followed the guide that I linked to above, it should also have made a web application as a client in Auth0 for you. We need to configure this, with a callback URL and some other settings.


Click on "Clients" in the left sidebar menu.

Find the client that was created, and click the "cog" icon at the right to access the settings.

![auth0clients.PNG]({{site.baseurl}}/_posts/auth0clients.PNG)


In here we want to notice the "Client ID", and the "Client Secret", which we need to store in our environment variables later on.

We also want to add our callback URL, which is the URL of the _omniauth_callbacks_controller.rb_ controller we will be creating very soon.

![auth0clientsettings.PNG]({{site.baseurl}}/_posts/auth0clientsettings.PNG)

Further down the page, there is a link for "Advanced Settings", click that, and find the "OAuth" tab.

We want to disable the "OIDC Conformant" for our scenario, because with this enabled, it will hide the values that would be shown in the results that we need from the NemID authentication service. This includes the CPR among other things.


![Auth0 OAUTH advanced settings]({{site.baseurl}}/_posts/auth0client.PNG)





### 2. Devise
Devise adds authentication support to our application.

#### Gemfile

Add "devise" to the Gemfile as shown below:

```ruby
gem 'devise'
```

Run "bundle install" to install the newly added gem.

#### Setup

To setup Devise, we need to run the install command, which will add the needed routes for our application and create the Devise initializer that we need to configure later on. 
We then want to create a Devise "User" model.

Run the following command:

```
rails g devise:install
```

Then:

```
rails g devise User
```

And lastly, migrate the database:

```
rake db:migrate
```

We should now have a working Devise installation with a "User" model for our application.



### 3. omniauth & omniauth-auth0
I followed the guide here for setting up the 2 gems

#### Gemfile

Add "omniauth" & "omniauth-auth0" to the Gemfile as shown below:

```ruby
gem 'omniauth'
gem 'omniauth-auth0'
```

Run "bundle install" to install the newly added gems.

#### Add columns to User model
We want to add 2 columns to our User model, "provider" and "uid".

Run the following command:
```
rails g migration AddOmniauthToUsers provider:string uid:string
```

And: 
```
rake db:migrate
```

#### Setup in User model

Locate the "User" model at:
```ruby
app/models/user.rb
```

Add the following to the list of devise modules:
```ruby
:omniauthable, omniauth_providers: %i[auth0]
```

Like so:
```ruby
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :trackable, :validatable, :omniauthable, omniauth_providers: %i[auth0]
```


Add the following two methods

```ruby
  def self.from_omniauth(auth)
    where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
      user.email = auth.info.email
      user.password = Devise.friendly_token[0,20]
      user.provider = auth.provider
      user.uid = auth.uid
      # If you are using confirmable and the provider(s) you use validate emails, 
      # uncomment the line below to skip the confirmation emails.
      # user.skip_confirmation!
    end
  end
  
  def self.new_with_session(params, session)
    super.tap do |user|
      if data = session["devise.auth0_data"] && session["devise.auth0_data"]["extra"]["raw_info"]
        user.email = data["email"] if user.email.blank?
      end
    end
  end
```


#### Setup in Devise initializer
Locate the Devise initializer at:
```ruby
config/initializers/devise.rb
```

Add the following: (Im using dotenv-rails for environment variables in development)
```ruby
config.omniauth :auth0, ENV['AUTH0_CLIENT_ID'], ENV['AUTH0_CLIENT_SECRET'], ENV['AUTH0_DOMAIN'], 
    :scope => 'openid'
```

_I placed it down where the OmnitAuth configuration is commented out_
![Devise initializer]({{site.baseurl}}/_posts/devise.PNG)





### 4. 

### 5. Create NemID test user

Go to https://appletk.danid.dk/testtools/ and login with user: "oces" and password: "nemid4all"

Click on "Autofill" at the very bottom.

Note down the "alias" and the "password"

Click "Submit".

You should now have a test NemID user.


## References & Inspiration
- [https://github.com/plataformatec/devise/wiki/OmniAuth:-Overview](https://github.com/plataformatec/devise/wiki/OmniAuth:-Overview)
- [https://www.digitalocean.com/community/tutorials/how-to-configure-devise-and-omniauth-for-your-rails-application](https://www.digitalocean.com/community/tutorials/how-to-configure-devise-and-omniauth-for-your-rails-application)
- [https://github.com/auth0/omniauth-auth0](https://github.com/auth0/omniauth-auth0)
- [https://auth0.com/docs/quickstart/webapp/rails](https://auth0.com/docs/quickstart/webapp/rails)
- [https://github.com/plataformatec/devise](https://github.com/plataformatec/devise)




