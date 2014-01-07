Date: 2013-09-12
Title: TempURL with CloudFiles/Swift
Author: Scott
Category: rackspace

CloudFiles and Swift has a neat feature that allows you to provide a temporary URL for users.  You can limit the URL by method and time.  This is useful if you have a file you want to allow someone to download temporarily, but not make completely public through the CDN.  *Note* My examples will use Rackspace CloudFiles, but you can do the same thing with any Openstack Swift setup, provided it has the TempURL middleware enabled in the pipeline.

Generating a TempURL requires a little bit of programming.  I will try and show how this can be done in a few languages.  The *first* step is to setup the TempURL Key on your account.  All that requires is setting the metadata for the account:

    curl -XPOST -H 'x-auth-token: <your auth token>' -H 'X-Account-Meta-Temp-Url-Key: sometempurlkey' http://<your storageURL>

That is all you have to do on the CloudFiles side.  The next part is figuring out which file you want to serve up with your TempURL.  Let's say you have a blogging service and CloudFiles hosts the content.  People could just right click on the photos and give the link to anyoen.  Basically using your service (and account) for photo hosting.  A TempURL to the image would be generated on every visith, but expire after a certain amount of time so that situation wouldn't happen.  Also helps if you just want to share a single object in a container without having to make the whole container CDN Enabled.  

The TempURL uses an HMAC key as the signature, generated from a string which consists of the method (GET, POST, etc), expiration in seconds and the object path:

    hmac_body = '%s\n%s\n%s' % (method, expires, object_path)

the object path would look something like:

    object_path = "/v1/MossoCloudFS_FooBar/my_tempurl_container/my_tempurl_object.txt"

the method would be:

    method = "GET"

the expiration in seconds (5 minutes):

    seconds = 60 * 5 

all together:

    import hmac
    from hashlib import sha1
    from time import time

    key = 'sometempurlkey'
    object_path = "/v1/MossoCloudFS_FooBar/my_tempurl_container/my_tempurl_object.txt"
    method = "GET"
    seconds = 60 * 5 
    expires = int(time() + seconds)
    hmac_body = '%s\n%s\n%s' % (method, expires, object_path)
    sig = hmac.new(key, hmac_body, sha1).hexdigest()
    url = '%s%s?temp_url_sig=%sAMP;temp_url_expires=%s' % \
        (<your storageURL basehref>, object_path, sig, expires)
            
or you could just use !(Swiftly)[http://gholt.github.io/swiftly/dev/]:

    swiftly tempurl GET /my_tempurl_container/my_tempurl_object.txt 600

