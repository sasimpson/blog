+++
Date = "2012-08-28"
Title = "Static Site with CloudFiles StaticWeb"
Author = "Scott"
categories = ["rackspace"]
Tags = ["rackspace"]
+++

One of the really cool features of CloudFiles is StaticWeb.  Essentially you can host whole _static_ site within a CloudFiles CDN enabled container.  I'm going to show you how to get a static site setup under CloudFiles.

You need to have a Rackspace Cloud account, which you can get [here](http://www.rackspace.com/cloud/).  Once you have your account setup, you can get your API key from the control panel.  Authenticating with Rackspace Cloud looks a little like:

    curl -XPOST https://identity.api.rackspacecloud.com/v2.0/tokens {"auth":{"RAX-KSKEY:apiKeyCredentials":{"username":"your rackspace cloud user","apiKey":"your rackspace cloud api key"}}}

which returns something like (i've removed some of the other bits):

    {
      "access": {
        "token": {
          "id": "auth token",
          "expires": "2012-08-20T12:00:00.000-00:00",
          ...
        },
        ...
        "serviceCatalog": [
          {
            "name": "cloudFilesCDN",
            "type": "rax:object-cdn",
            "endpoints": [
              {
                "region": "DFW",
                "tenantId": "MossoCloudFS_accounthash",
                "publicURL": "https://cdn1.clouddrive.com/v1/MossoCloudFS_accounthash"
              },
              ...
            ]
          },
          {
            "name": "cloudFiles",
            "type": "object-store",
            "endpoints": [
              {
                "region": "DFW",
                "internalURL": "https://snet-storage101.dfw1.clouddrive.com/v1/MossoCloudFS_accounthash",
                "tenantId": "MossoCloudFS_accounthash",
                "publicURL": "https://storage101.dfw1.clouddrive.com/v1/MossoCloudFS_accounthash"
              },
              ...
            ]
          },
          ...
        ]
      }
    }

You need a few parts of this output.  First, the cloudFiles publicURL, which is your CloudFiles API endpoint.  Second, is the cloudfFilesCDN publicURL and last the id in the token section.  With your auth token and the storage endpoint you can create a container for your site.  For example:

    curl -XPUT -H 'x-auth-token: your-rackspace-auth-token' <cloudFiles:publicURL>/blog

This will create the container for your site.  Next you need to CDN Enable your container.  Using the cloudFilesCDN publicURL you want to CDN Enable the container:

    curl -XPUT -H 'x-auth-token: your-rackspace-auth-token' -H 'x-ttl: 900' <cloudFilesCDN:publicURL>/blog

The **x-ttl** header will set the Time to Live to 900 seconds, or 15 minutes, the lowest you can have it.  The TTL or Time To Live header tells the CDN edge how long to cache your files for.  I set mine to 15 minutes so that if and when I make changes to my site it will only live for 15 minutes.  Once you have a container, and its public, you have to set the **X-Container-Meta-Web-Index** header to whatever your index file is called, usually index.html or index.htm:

    curl -XPOST -H 'x-auth-token: your-rackspace-auth-token' -H 'x-container-meta-web-index: index.html' <cloudFiles:publicURL>/blog

Now you just need to put your site up and everything should work great.  Your CDN URL, or what you can visit to see your site should be in the headers when you do a HEAD on the CDN management URL:

    curl -XHEAD -H 'x-auth-token: your-rackspace-auth-token' <cloudFilesCDN:publicURL>/blog

which should give you something like (this blog for example):

    HTTP/1.1 204 No Content
    X-Ttl: 900
    X-Cdn-Uri: http://acd5dc0bfb0c09101f5b-8902cd1ab3d0222505a9800f528c1a8b.r67.cf1.rackcdn.com
    X-Cdn-Enabled: True
    X-Log-Retention: False

and you might see some extra URI's in there but you want the **X-CDN-URI** header, this is what you visit to see your site.  You can use a DNS CNAME to make this easier to remember.

**note** CloudFiles now has multiregion support for the US.  You can create containers and objects in multiple regions (DFW and ORD for now), which means that you need to pay attention to which endpoints you use from the auth call.

**note 2** (updated 8/7/13) if you want to use this in another region, you need to grab the cloudFiles:publicURL from the region you want.  in my Auth 2.0 example above, i show the DFW region, but you could just as easily use ORD or SYD.  The endpoints are all listed in the Auth 2.0 return data, but really you just need to substitue ord1 or syd2 for dfw1 in your endpoint.
