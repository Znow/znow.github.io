---
published: false
layout: post
categories: Ruby on Rails
---
## Integrate TinyMCE editor in RefineyCMS

I have been using RefineryCMS in a client project for quite some time. And the standard "build-in" wysiwyg editor "WYMeditor" is not very up to date, neither working properly.

TinyMCE is another wysiwyg editor, which is a very nice alternative, simple and functional.

You can integrate TinyMCE in your solution simply by adding these gems:
```ruby
gem 'tinymce-rails'
gem 'tinymce-rails-langs'
gem 'tinymce-rails-imageupload', '~> 4.0.16.beta'
```
(and of course removing 'refinerycms-wymeditor' from the Gemfile)

In RefineryCMS, with WYMeditor enabled, you had the possibility to add images from the "Image"-archive tab. But what if you want to use this feature with the TinyMCE editor?







