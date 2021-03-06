---
layout: post
title: Ruby on Rails - AJAX pagination with Kaminari
published: true
categories: Ruby on Rails
---

I needed a pagination feature, without the need of a full page reload. So I had a look at Kaminari, which is the only paging gem that seems to be active. All though, the default implementation of Kaminari with AJAX enabled by just doing:

{% highlight haml linenos %}
= paginate @users, :remote => true
{% endhighlight %}

Didn't seem to work at all...

Here's my implementation which is fully working. I'm using news articles as an example.

_news_controller.rb_

{% highlight ruby linenos %}
def index
  @news = News.all.page(params[:page]).per(10)

  respond_to do |format|
      format.html
      format.js { render :layout => false }
  end
end
{% endhighlight %}


_ _news.haml_

{% highlight haml linenos %}
  %table.table.table-condensed.table-striped.table-hover
    %tr
      %th Title
      %th Content
    - @news.each do |n|
      %tr
        %td= n.title
        %td= n.content
{% endhighlight %}

_index.haml_

{% highlight haml linenos %}
  #news-table
    = render 'news'

  #news-paginator
    = paginate @news, :remote => true

{% endhighlight %}

_index.js.erb_

{% highlight erb linenos %}
  $('#news-table').html('<%= escape_javascript(raw(render 'news')) %>');
  $('#news-paginator').html('<%= escape_javascript(paginate(@news, remote: true)) %>');
{% endhighlight %}

Hope this implementation works for those that faced the same problem as I, with the default AJAX pagination not working.

Feel free to PM me if any questions.

Regards
