---
layout: post
title: "Get blogging with Octopress on Heroku"
date: 2012-01-14 20:01
comments: false
categories: [blogging, heroku]
---

Following on from [my opening post]({{ site.url }}/blog/2012/01/14/hello-2012/), one of the main hurdles preventing me from blogging was the setup of a blogging platform and the management of the hosting. Thankfully, [Octopress](http://octopress.org) and [Heroku](http://heroku.com) have provided a simple, effective and free way for me to publish my thoughts to the web. In this post I will explain how I got started with Octopress on Heroku.

<!-- More -->

## Setting up your development environment

First off you are going to require **Ruby 1.9.2**. At this point your natural temptation may be to [download and compile the source code](http://www.ruby-lang.org/en/downloads/) directly. This will work for the most part, but any seasoned Ruby developer will advice you against this path.

When doing any serious Ruby development you most likely will be working on numerous projects that use a variety of different frameworks, each with their own specific Ruby and [Gem](http://rubygems.org/) version requirements. Managing multiple versions of Ruby and Gems manually is a rather daunting task. 

Thankfully this problem has been solved already by [RVM](http://beginrescueend.com/rvm/install/) and [Rbenv](https://github.com/sstephenson/rbenv). Both RVM and Rbenv provide an isolated Ruby environment, encapsulating Ruby and Gems to specific versions. This allows each Ruby project on your system to run a targeted version of Ruby and Gems per project without concerning yourself with conflicts. I am not going to advocate either of these tools over each other, but I will be using RVM in this article. There are instructions on using Rbenv on the Octopress [setup guide](http://octopress.org/docs/setup/).

To install **RVM**, in Terminal run;

```
$ bash < <(curl -s https://rvm.beginrescueend.com/install/rvm)
```

Once **RVM** has finished installing, you should add it to your shell as a command. Run the following command to update your `.bash_profile`.

```
$ echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && . "$HOME/.rvm/scripts/rvm" # Load RVM function' >> ~/.bash_profile
```

Finally you should update your Terminal sessions `.bash_profile` to load the new profile settings.

```
$ source ~/.bash_profile
```

Now you're ready to install Ruby 1.9.2.

```
$ rvm install 1.9.2 && rvm use 1.9.2
```

This command installs Ruby 1.9.2 as a Gem into your `~/.rvm/gems/` folder. Now within each Ruby application, you can create a `.rvmrc` file that defines the Gem sets required by the project. For example, Octopress instructs _RVM_ to use Ruby 1.9.2 by including the following in its `.rvmrc` file.

```
rvm use 1.9.2
```

## Obtaining Octopress

The best way of getting Octopress is directly from [Github](http://github.com//imathis/octopress). This will require your system to have [Git](http://git-scm.com) installed. By cloning the Octopress project, obtaining updates in the future will be as easy as `git pull origin master`. However you may prefer to fork the project on Github and then clone from your own repository. In this case, I am happy with the standard build of Octopress, so will clone it directly.

```
$ git clone http://github.com/imathis/octopress.git ~/Projects/octopress
```

This will place an instance of Octopress into my `~/Projects/octopress` folder. You may wish to place it elsewhere, it does not matter where at this time.

Now change directory into the newly created local Git repository and check the Ruby version.

```
$ cd ~/Projects/octopress
$ ruby --version
ruby 1.9.2p290 (2011-07-09 revision 32553) [x86_64-darwin11.2.0]
```

You should see the Ruby version you previously installed, the exact version number and details will differ depending on your platform and other environment variables. The Ruby environment is controlled by the RVM  `~/Projects/octopress/.rvmrc` file I mentioned earlier.

Next we install [Bundler](http://gembundler.com/) into this project.

```
$ gem install bundler
$ bundler install
```

Taken from its own website, Bundler provides the following functionality;

{%blockquote%}
Bundler manages an application's dependencies through its entire life across many machines systematically and repeatably.
{%endblockquote%}

When the `bundler install` command is invoked, it reads the required list of Gems from the projects `Gemfile`. If you were inspect the `Gemfile` for Octopress, you can see the project dependencies.

Finally, to install the default Octopress theme run the following command.

```
$ rake install
```

This will setup your local instance of Octopress with the standard theme. In my instance this command failed with the following message.

```
$ rake install
rake aborted!
You have already activated rake 0.9.2.2, but your Gemfile requires rake 0.9.2. Using bundle exec may solve this.

(See full trace by running task with --trace)
```

In some circumstances this will happen if you have a newer version of rake installed for whatever reason. Usually the Gemfile and Bundler will resolve these issues, but because both `rake 0.9.2` and `rake 0.9.2.2` are matched in the Gemfile this conflict is not resolved. However in this case, following prescribed advice provides the expected outcome.

```
$ bundle exec rake install
## Copying classic theme into ./source and ./sass
```

Octopress is now installed locally and is ready for you to begin blogging. You can use a plethora of hosting services with Octopress. As Octopress compiles a static version of your site, it would be perfectly possible just to publish the `public/` folder to any standard HTTP server such as Apache or Nginx.

Because the contents of `public/` folder is compiled by Octopress, by default the Git clone of the project will contain a `.gitignore` file with an entry to ignore the `public` folder. We need to remove this entry to ensure the `public/` folders contents are included in our git repository and pushed upstream.

To remove the entry, edit the `.gitignore` file remove the offending line.

## Deployment to Heroku

{% img http://f.cl.ly/items/0A0C3C1V173U2j0q213S/Image%202012.01.14%2021:22:21.png Heroku Homepage %}

Deployment to Heroku requires an account on Heroku. If you do not have an existing Heroku account, [sign up](https://api.heroku.com/signup) for one now.

I am discovering that as with most things in Ruby, there is usually a Ruby Gem tailored to working with whatever Ruby service you require. Heroku is no different.

To install the Heroku Gem.

```
$ gem install heroku
```

Now we have the Heroku Gem available, we can create our Heroku app. This does require an existing Heroku account, so if you still have not signed up with the Heroku service, now really is the time!

**Note:** To proceed you will need to have a SSH public key. Github have a [great guide](http://help.github.com/set-up-git-redirect/) if you require help.

First we need to tell Heroku about our account. Using the `heroku login` command, we can authenticate ourselves with the Heroku service.

```
$ heroku login
Enter your Heroku credentials.
Email: <your email address>
Password: <your password>
Found existing public key: /Users/<your user>/.ssh/id_rsa.pub
Uploading ssh public key /Users/<your user>/.ssh/id_rsa.pub
```

If successful, the `heroku login` command will upload our SSH public key automatically to the Heroku service. The reason for this is that you publish to Heroku using Git, and just like any other standard Git remote repository, connection is made over SSH.

Next we create an application on Heroku that will host this blog.

```
$ heroku create
Creating smooth-beach-2208... done, stack is bamboo-mri-1.9.2
http://smooth-beach-2208.heroku.com/ | git@heroku.com:smooth-beach-2208.git
```

Well we have an app, but I don't like the name. Heroku will generate a random name for applications if no name is supplied. After doing some research, I discovered you can provide a name, thus `heroku create defreyssinet` would result in an application located at `defreyssinet.heroku.com`. However it is not a disaster as we can rename any Heroku application after creating it.

To rename our app, within the `~/Projects/octopress` folder type.

```
$ heroku rename defreyssinet
http://defreyssinet.heroku.com/ | git@heroku.com:defreyssinet.git
Git remote heroku updated
```

This has updated the hostname on Heroku and rather neatly tidied up our local Git repository remote entries. It is possible to do this manually by logging on to the Heroku site and renaming the app there, then updating the local git repository remote entries.

```
$ git rm heroku
$ git remote add heroku git@heroku.com:defreyssinet.git
```

At this point we have a Heroku app linked to our local Octopress Git repository. We can verify that our app has indeed been created by logging onto Heroku and selecting the *My Apps* menu item.

{% img  http://f.cl.ly/items/3c1p3l021j2w0V011Y1I/Image%202012.01.14%2022:52:15.png Heroku Applications %}

Now we have a local instance of Octopress and a host ready to accept our new blog. What is lacking is the content.

## Creating a new post

As I mentioned earlier, Octopress is a static site generator rather than a fully fledged web application like _Wordpress_ or _Habari_. This implies that there is a specific workflow in relation to publishing using Octopress. The typical publication flow for an article is as follows;

 1. Create an article - `rake new_post["title"]`
 2. Write some content
 3. Review - `rake generate`
 4. _If more work required, return to step 2._
 5. Publish - `git push heroku <your app>`

Before we get ahead of ourselves, it is time to setup the configuration of the blog using the `_config.yaml` file found in the root of the project. All of the configuration settings are described in detail on the [Configuring Octopress](http://octopress.org/docs/configuring/) page. Before creating any content it is probably worth familiarizing and editing this file to your specific needs.

Creating an article simply uses the `rake new_post["title"]` task. This creates a new file in the `~/Projects/octopress/source/_posts/` folder with a name composed of the date and the title. The extension will be `.markdown` as we will be writing in [Markdown](http://daringfireball.net/projects/markdown/) format.

**Note:** We will only be working in the `source/` folder and we *must* not touch the `public/` folder. The `rake generate` task compiles the contents of `source/` and places it into `public/`.

So lets go ahead and create an article. (*I'm prefixing with `bundle exec` here to avoid the version complaint seen earlier.*)

```
bundle exec rake new_post["Get blogging with Octopress on Heroku"]
```

Rake helpfully creates a new Markdown file called `2012-01-14-get-blogging-with-octopress-on-heroku` in the `source/_posts` folder. Opening up the new file will reveal the following.

{%codeblock source/_posts/2012-01-14-get-blogging-with-octopress-on-heroku%}
---
layout: post
title: "Get blogging with Octopress on Heroku"
date: 2012-01-14 12:24:32
comments: true
categories:
---
 
{%endcodeblock%}

The header of this file contains some [yaml front matter](https://github.com/mojombo/jekyll/wiki/yaml-front-matter). There are additional options available to control the publication state and author. ~~Personally I think the time of publication is less important, but this is easily fixed by removing the time index from the `date:` entry~~ (**Edit:** don't do this as it will mess up article ordering!). I have also turned off comments `comments: false` for this post.

It is now possible to add some content to this page in Markdown format. We don't need a title as the filename will become the title, so don't start with a `# my  title` block.

{%codeblock source/_posts/2012-01-14-get-blogging-with-octopress-on-heroku%}
---
layout: post
title: "Get blogging with Octopress on Heroku"
date: 2012-01-14
comments: false
categories: [blogging, heroku]
---

Following on from [my opening post](http://def.reyssi.net/blog/2012/01/14/hello-2012/), one of the main hurdles preventing me from blogging was the setup of the blog platform and the management of the hosting. Thankfully, [Octopress](http://octopress.org) and [Heroku](http://heroku.com) have provided a simple, effective and free way for me to publish my thoughts to the web. In this post I will explain how I got started with Octopress on Heroku.

...
{%endcodeblock%}

With some content in place, we can generate our first blog post using the `rake generate` task.

```
$ bundle exec rake generate
## Generating Site with Jekyll
unchanged sass/screen.scss
Configuration from /Users/sdefreyssinet/Projects/octopress/_config.yml
Building site: source -> public
Successfully generated site: source -> public
```

At this point Octopress has compiled the static version of our site. It is possible to load the `public/index.html` folder into any browser to view our new content. 

If like me you are using Mac OS X, you can use the [POW](http://pow.cx/) zero-configuration rack server from [37signals](http://37signals.com/) to host your site locally.

With POW installed, the following commands will create a local web server hosting our Octopress site.

```
$ cd ~/.pow
$ ln -s ~/Projects/octopress octopress
```

Now it is simply a case of navigating to [http://octopress.dev](http://octopress.dev) to see the site we just compiled. This is a really quick and simple way of reviewing changes.

*Users on other platforms that wish to test in a server environment locally just need to host their local `public/` folder.*

Previewing the site should present a page that looks similar to the screenshot below.

{% img http://f.cl.ly/items/0a392t0q132I340W2s2Z/Image%202012.01.15%2000:43:09.png 'Our first post hosted by POW' %}

Looking good. At this point we can carry on editing and reviewing until we're happy the post is ready. Remember that each time a change is made, an additional `rake generate` task needs to be invoked before any changes will be available in the `public/` folder.

## Publishing to Heroku

The first post is completed, reviewed and ready for publication. This is where Heroku makes our lives very easy. First we need to check all of our changes into the local Git repository.

```
$ git add .
$ git commit -am 'Initial commit with our first post, get blogging with octopress on heroku'
```

With the local repository up to date, we can push the repository to Heroku. Once the repository is pushed to Heroku it will be available *immediately*, so be sure you're *really* ready to publish before pushing upstream.

```
$ git push heroku master
Counting objects: 77, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (39/39), done.
Writing objects: 100% (49/49), 4.80 KiB, done.
Total 49 (delta 26), reused 0 (delta 0)

-----> Heroku receiving push
-----> Ruby/Sinatra app detected
-----> Gemfile detected, running Bundler version 1.0.7
       All dependencies are satisfied
-----> Compiled slug size is 784K
-----> Launching... done, v8
       http://def.reyssi.net deployed to Heroku

To git@heroku.com:defreyssinet.git
   efce197..2f3ed54  master -> master
```

Once the repository is pushed upstream, it will be available from the address configured in the Heroku app. In this case we named the app `defreyssinet` so the site will be available from [http://defreyssinet.heroku.com](http://defreyssinet.heroku.com).

## In conclusion

In this blog post we have covered how to create a blog using the Octopress static site generator and host it on Heroku. In particular we have covered provisioning a local development environment with POW, obtaining Ruby using RVM and cloning the Octopress project from Github. We created a new Heroku application instance using the Heroku cli interface. Then we discovered how the publishing cycle works with Octopress, revolving around the `rake generate` task. Finally we pushed the Octopress blog to Heroku for production.

This article only scratches the surface of what Octopress offers, for more information I encourage everyone to read the official  [documentation](http://octopress.org/docs). 

Octopress will certainly not be to everyones tastes, but I hope this article has provided those with the slightest curiosity about Octopress some impetus to try it from themselves.