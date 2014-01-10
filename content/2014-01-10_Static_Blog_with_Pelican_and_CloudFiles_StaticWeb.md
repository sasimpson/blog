Date: 2014-01-10
Title: Static Blog with Pelican and CloudFiles Static Web
Author: Scott
Category: rackspace

I've wanted to post up how I make this blog work for a while now.  It seemed like a daunting task, but after I started [another one](http://blog.kf5way.com) for my ham radio stuff, I realized it wasn't that hard.

You can apply this to any static host, but we're going to host it on CloudFiles.  CloudFiles is $.10/GB, and I know my blog is less than a GB.  So for cents a month you can host an entire blog.  Pelican gives you themes you can use, or create your own.  You can use markdown, restructured text or asciidoc.  I use markdown since it is easy to format and pretty standard (you can use for github docs).  

First step is to get [Pelican](http://blog.getpelican.com).  Pelican is a python project and you need at least Python 2.7. I use python virtualenvs for things like this.  If you don't have virtualenv installed you can check out the docs [here](http://www.virtualenv.org/en/latest/) or simpler with [virtualenvwrapper](http://virtualenvwrapper.readthedocs.org/en/latest/).  Install pelican in your virtualenv or not:

    pip install pelican

This will install pelican and all of its dependencies.  Next you need to make the directory for your blog to reside locally:

    mkdir ~/Projects/python/test/blog

At this point if you want to use git to do version control or store your blog on github:

    cd ~/Projects/python/test/blog; git init

Then init your pelican blog:

    pelican-quickstart

This will help you get started by answering a few questions.  You can use the CloudFiles support, but currently this will only put the blog into your default region.  If you don't care, then go for it.  Plug in your username and key.  You end up with a nice directory structure and all kinds of files.  One file that is very important is the pelicanconf.py.  Here is what it looks like after being generated:

    #!/usr/bin/env python
    # -*- coding: utf-8 -*- #
    from __future__ import unicode_literals

    AUTHOR = u'Scott'
    SITENAME = u'test blog'
    SITEURL = ''

    TIMEZONE = 'Europe/Paris'

    DEFAULT_LANG = u'en'

    # Feed generation is usually not desired when developing
    FEED_ALL_ATOM = None
    CATEGORY_FEED_ATOM = None
    TRANSLATION_FEED_ATOM = None

    # Blogroll
    LINKS =  (('Pelican', 'http://getpelican.com/'),
              ('Python.org', 'http://python.org/'),
              ('Jinja2', 'http://jinja.pocoo.org/'),
              ('You can modify those links in your config file', '#'),)

    # Social widget
    SOCIAL = (('You can add links in your config file', '#'),
              ('Another social link', '#'),)

    DEFAULT_PAGINATION = 10

    # Uncomment following line if you want document-relative URLs when developing
    #RELATIVE_URLS = True

Adjust your timezone, url, etc.  To make this into a site, you just type:

    make html

and your blog will be generated and will be in the ./output directory.  You can simulate it by using:

    make serve

and go to [http://localhost:8000](http://localhost:8000) to view it.  Of course this probably won't work because you have no content!  You can create posts in the ./content directory.  I use markdown so my posts end with .md.  I like to organize like:

    ./content/2013-07-11_Some_Cool_Post.md
    ./content/2013-07-12_Some_Cool_Post_2.md

but you can do it however you like.  That method helps them be sorted easier.  Run `make html` again and you should have the posts in ./content generated in output now!

Now where to put it?  You can put this where ever, but CloudFiles is cheap and easy.  Follow [this](http://blog.scottic.us/static-site-with-cloudfiles-staticweb.html) post on how to get staticweb and the CDN working for a container.  Then if you used the CloudFiles support on Pelican, you just type

    make cf_upload

and that should upload your output directory to the container you specified.  If you want to use [swiftly](http://gholt.github.com/swiftly), first install it (I install it globally, not in a virtualenv, since I use it for everything):

    pip install swiftly

then:

    swiftly -A https://identity.api.rackspacecloud.com/v2.0 -U username -K apikey put -i output <blog_container>

Go to your cdn url and you should see your blog!  Pelican has support for Disqus too, so you can have a static blog that has commenting.  Check out their [documentation](http://docs.getpelican.com) on all their features and themes.  Questions or comments can be left below.

