---
title: "Using Hugo with GCP Storage and CircleCI"
date: 2019-02-06T13:45:59-08:00
---

I have been usinag a combination of Hugo+GitHub+Wercker+CloudFiles to compose, store, build, and host my blog for a while.  Recently, my wercker integration broke and wercker was also purchased by Oracle.  I have been pretty familiar with CircleCI for a while now, so I decided to give that a try for this process.  I also no longer (2 years) work for Rackspace, so I decided to move my blog to Google Cloud Platform's Storage service and use it for my static hosting needs.  

I was using another image for doing hugo builds on wercker, but it hadn't been updated, so I build my own, [here](https://github.com/sasimpson/hugo), which is a pretty simple dockerfile.

Essentially, an alpine image with the hugo binary added to it in an executable spot.  

CircleCI is the magic bits that grab that image, then build the hugo content out into static content that needs to be uploaded.  In my CircleCI config (which you can view in the repo under [.circleci/config.yml](https://github.com/sasimpson/blog/blob/master/.circleci/config.yml)) you can see I've leveraged their Persistent Workspaces feature.  This lets you persist files across images in a "workspace".  The major advantage of this is instead of storing the files as artifacts then having the second job download those artifacts, we just tell CircleCI to persist that workspace across jobs.  

The second job does the deploy bits, using the Google Cloud SDK image and injecting our credentials via environment variables stored in CircleCI, then using gsutil to upload the content to the bucket.  

So with a hugo blog, such as this one, you just need to have the .circleci config directory with the config similar to mine then setup your circleci project and this will automate the build and deploy of the blog.  

### Dockerfile Breakdown

Here is the Dockerfile for the Hugo image:

    # FROM golang:1.11-alpine as builder
    FROM alpine:latest

    # get updates and install necessary packages for cicleci
    RUN apk update && apk add wget && apk add git && rm -rf /var/cache/apk/*

    RUN wget https://github.com/gohugoio/hugo/releases/download/v0.54.0/hugo_0.54.0_Linux-64bit.tar.gz
    RUN tar -C /usr/bin -zxf hugo_0.54.0_Linux-64bit.tar.gz

Broken down, base image is alpine latest and we use the alpine package manager to update and get wget, which we need to get the binary from the hugo release:

    # FROM golang:1.11-alpine as builder
    FROM alpine:latest

    # get updates and install necessary packages for cicleci
    RUN apk update && apk add wget &&  rm -rf /var/cache/apk/*

Next we run wget to get the release binary:

    RUN wget https://github.com/gohugoio/hugo/releases/download/v0.54.0/hugo_0.54.0_Linux-64bit.tar.gz

Then extract the binary into the /usr/bin directory so our circleci runner can execute it:

    RUN tar -C /usr/bin -zxf hugo_0.54.0_Linux-64bit.tar.gz

### CircleCI Config Breakdown

CircleCI config (.circleci/config.yml):

    version: 2
    jobs:
      build:
        docker:
        # specify the version
        - image: ssimpson/hugo:latest

        steps:
        # get the code
        - checkout

        # update the theme submodule
        - run: git submodule update --init themes/terminal
        # - run: git submodule update --recursive --remote

        - run: hugo version
        #run hugo to generate with theme
        - run: hugo -t terminal

        #upload artifacts for next job step
        # - store_artifacts:
        #     path: /root/project/public

        - persist_to_workspace:
            root: /root/project
            paths:
            - public
    
      deploy:
        docker:
        - image: google/cloud-sdk
        
        steps:
        - attach_workspace:
            at: /tmp/project

        - run: | 
            echo $GCLOUD_SERVICE_KEY | gcloud auth activate-service-account --key-file=-
            gcloud --quiet config set project ${GOOGLE_PROJECT_ID}
            gcloud --quiet config set compute/zone ${GOOGLE_COMPUTE_ZONE}
        - run: gsutil -m cp -r /tmp/project/public/* gs://blog.scottic.us/

    workflows:
      version: 2
      build_and_deploy:
        jobs:
        - build
        - deploy:
            requires:
                - build

CircleCI config first has you declare a version, then your jobs, then you define a workflow using the jobs we declared.  You can declare the jobs without a workflow, but then you cannot easily define the dependency of one on the other (at least this is what I found).  Also, the YAML formatting might be not great here, so use CircleCI's local runner to test these.

We have two jobs, build:

    jobs:
      build:
        docker:
        # this is the hugo image made above
        - image: ssimpson/hugo:latest

        steps:
        # get the code/blog content
        - checkout

        # update the theme submodule, you have to do this as the normal git checkout will not update the submodule.
        - run: git submodule update --init themes/terminal

        #run hugo to generate
        - run: hugo

        #persist workspace so that the deploy setup can use what we just generated
        - persist_to_workspace:
            root: /root/project
            paths:
            - public

then deploy:

      deploy:
        docker:
        # this is an image maintained by circleci
        - image: google/cloud-sdk
        
        steps:
        # attach the existing workspace to the /tmp/project dir
        - attach_workspace:
            at: /tmp/project

        # login to gcloud using the auth key saved in the env variable
        - run: | 
            echo $GCLOUD_SERVICE_KEY | gcloud auth activate-service-account --key-file=-
            gcloud --quiet config set project ${GOOGLE_PROJECT_ID}
            gcloud --quiet config set compute/zone ${GOOGLE_COMPUTE_ZONE}

        # upload our content
        - run: gsutil -m cp -r /tmp/project/public/* gs://blog.scottic.us/

The workflow lets us declare dependencies, such as deploy needing build to be successful to continue:

    workflows:
      version: 2
      build_and_deploy:
        jobs:
        - build
        - deploy:
            requires:
                - build

I will work on making more of this stuff less specific to my need with environment variables or something.  Any questions, hit up on twitter: @scotty
