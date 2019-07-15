+++
title = "Using Wercker to deploy Hugo to Cloud Files"
date = "2017-01-07T19:36:56-06:00"

+++

I've been using Hugo for a while now, on this blog, and my ham radio blog at [blog.ke5eo.com](http://blog.ke5eo.com).  One draw back to how I've had this configured is that I need to generate the content on my computer, usually my MacBook, then upload using [swiftly](https://github.com/gholt/swiftly).  Swiftly is a great tool for this because Hugo cranks out the content to a directory called "public" and I can use the -i flag on swiftly to upload an entire directory to a container like so:

    swiftly -A https://identity.api.rackspacecloud.com/v2.0 -U username -K apikey put -i public <blog_container>

which isn't unlike what I was doing for Pelican.  However, my windows desktop I use at home doesn't quite have the flexibility for using swiftly, i.e. it's kind of a pain in the rear.  So I took a look at how the Hugo folks suggest doing it, and their example used Wercker to upload to GitHub pages.  The first bit, generation, was fine for my purposes, but I wanted to use swiftly.  So in my root dir for my Hugo blog, I created a **wercker.yml** file to perform this task:

    box: busybox
    build:
        box: google/golang
        steps:
            - script:
                name: initialize git submodules
                code: |
                    git submodule update --init --recursive
            - arjen/hugo-build:
                version: "0.18.1"
                theme: vienna
                disable_pygments: true
                flags: -v
    deploy-to-cloudfiles:
        box: python:2.7
        steps:
            - pip-install:
                requirements_file: ""
                packages_list: "swiftly six"
            - script:
                name: call swiftly
                code: swiftly --verbose --region $WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_REGION put -i ./public $WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_CONTAINER 

The build part is almost the same, other than I included a piece to make sure the git submodules are up to date since I use that for themes.

    - script:
        name: initialize git submodules
        code: |
            git submodule update --init --recursive

I found that if I didn't do this, it didn't even have a default theme to generate the content and didn't really work.  Next is the *deploy-to-cloudfiles* pipeline.

    deploy-to-cloudfiles:
        box: python:2.7
        steps:
            - pip-install:
                requirements_file: ""
                packages_list: "swiftly six"
            - script:
                name: call swiftly
                code: swiftly --verbose --region $WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_REGION put -i ./public $WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_CONTAINER    

Since the Hugo build stuff uses a go based container, the *box* directive tells it to use a python container instead.  Then we need to use pip to install swiftly and then lastly call swiftly to do the upload.  The environment variables can be set in Wercker, the ones such as $SWIFTLY_AUTH_URL and $SWIFTLY_AUTH_USER.  I used WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_REGION and WERCKER_RACKSPACE_CLOUDFILES_SWIFTLY_CONTAINER to do the region and container.  

With wercker you can create the *wercker.yml* file in your project, then setup the app within wercker and it will build automatically.  The deploy pipeline has to be added and chained to the build pipeline in the workflows screen.  

I think for the 2 concurrent jobs this is a pretty easy way to do deploys, however, I do wish they had some hobbist pricing, as their regular pricing is $350/mo for 3+ concurrent jobs.  I don't really plan on having a bunch of concurrent jobs, so this will work just fine.
