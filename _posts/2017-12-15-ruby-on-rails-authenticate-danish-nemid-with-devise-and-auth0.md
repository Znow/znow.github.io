---
published: false
---
## Ruby on Rails - Authenticate Danish NemID with Devise and Auth0

### Introduction

A project that I worked on required the option to let users sign in with their "e-ID", the Danish NemID login service.
It's used for signing in to online banking, social customer service and other stuff that require validation of the users identity.

I used alot of time on researching different possibilities, on how to implement NemID in Ruby on Rails - and there isn't really any easy plug-and-play solution, yet.

[Criipto](https://criipto.com/) (former Grean) is a supplier of a service that lets you use NemID together with [Auth0](https://auth0.com/) - and Auth0 is used for SSO(Single Sign On), and can be integrated in many languages. Criipto also has the options for use of Norwegian and Swedish login services.

Auth0 made a very fine starting point [here](https://auth0.com/authenticate/rails/nemid/) on how to get up and running with their and Criipto's service.

But it's not really enough, to get it working in a Rails app, so I will try to outline that in the following steps, with what I did to get everything working with Devise aswell.

### Prerequisites

Make sure you have a working Rails app, atleast at Rails 4.2.10.

The tutorial have been tested with Rails 4.2.10 and Rails 5.1.4, Ruby 2.4.3 on RVM.

### 1. Register with Criipto and Auth0

Go to https://auth0.com/authenticate/rails/nemid/ and follow the guide.

### 1.1

Once you have done the 

### 2. Create new Ruby on Rails app

### 3. Setup Devise

### 4. Setup omniauth, omniauth-auth0

### 5. Create NemID test user

Go to https://appletk.danid.dk/testtools/ and login with user: "" and password: ""

Click on "Autofill" at the very bottom.

Note down the alias and the password

Click "Submit".


