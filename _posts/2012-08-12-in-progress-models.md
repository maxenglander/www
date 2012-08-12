---
layout: default
title: Make image uploads resilient to form redisplays
---

I'm using [Paperclip](http://github.com/thoughtbot/paperclip) on Rails, and ran into this issue: when submitting a POST request to Rails, image data is not resilient to form redisplays. So, if a user tries creating a new model (with an image attachment), and forgets to fill out a required field (like `description`), Rails will drop the image data. The user must *choose and re-submit* the image in the next POST.

The top answer to [this StackOverflow question](http://stackoverflow.com/questions/5198602/not-losing-paperclip-attachment-when-model-cannot-be-saved-due-to-validation-err) recommends switching to [Carrierwave](https://github.com/jnicklas/carrierwave/), an alternative to Paperclip which sports a nifty local caching feature.

I didn't feel like this was a good solution to me for a couple of reasons:

 1. The issue is small and easily fixable without swapping a critical code path
 2. I'm on Heroku, which makes taking advantage of Carrierwave's local caching feature [problematic](http://rickenharp.posterous.com/using-carrierwave-caching-on-heroku)

I decided to set up conditional validations on `Art`. First, I add an `is_complete_flag` to my `Art` model:

{% highlight ruby %}
    class AddIsCompleteFlagToArts < ActiveRecord::Migration
        def up
            add_column :arts, :is_complete_flag, :boolean, default: true, null: false
        end
    end
{% endhighlight %}

In my model, this:

{% highlight ruby %}
    class Art < ActiveRecord::Base
        has_attached_file :image

        validates_presence_of :title
        validates_attachment_presence :image
    end
{% endhighlight %}

Becomes this:

{% highlight ruby %}
    class Art < ActiveRecord::Base
        has_attached_file :image

        validates_presence_of :title, unless: :in_progress
        validates_attachment_presence :image

        before_save do
           self.is_complete_flag = self.errors.empty?
        end

        def progressive do |self|
            self.valid?
            self.in_progress = true
            yield self
            self.in_progress = false
        end

        protected

        attr_accessor :in_progress
    end
{% endhighlight %}

From `ArtsController`, I do this:

{% highlight ruby %}
    class ArtsController < ApplicationController
        def create
            @art = Art.new params[:art]
            @art.progressive(&:save)

            if @art.complete?
                redirect_to @art
            else
                render :edit
            end
        end

        def edit
            @art = Art.find params[:id]  
        end

        def update
            @art = Art.find params[:id]

            @art.progressive(&:save)

            if @art.complete
                redirect_to @art
            else
                render :edit
            end
        end
    end
{% endhighlight %}

Some other things I might do to build on this:
- Add a `completed` scope to my `Art` model to only return completed art from queries
- Add a scheduled task to periodically clear out incomplete art
