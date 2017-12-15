---
published: false
---
## Ruby on Rails - Authenticate Danish NemID with Devise and Auth0

### Introduction
A project that I work required the option to let users sign in with their "eID", the Danish NemID.
It's used for signing in to online banking, social customer service and other stuff that require validation of the users identity.

I used alot of time on researching different possibilities, on how to implement NemID in Ruby on Rails - and there isn't really any easy plug-and-play solution, yet, neither from NETS side.

### Prerequisites
Make sure you have a working Rails app, atleast at Rails 4.2.10.


1. Follow https://auth0.com/authenticate/rails/nemid/
2. Create new Ruby on Rails app
3. Setup Devise
4. Setup omniauth, omniauth-auth0
5. 
