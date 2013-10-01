Date: 2013-10-1
Title: Role Based Access Control and CloudFiles
Author: Scott
Category: programming,openstack,rackspace

Recently Rackspace added Role Based Access Control to the identity system.  This means the main account user can create sub users and roles and that main user can allow the sub users to have access to portions of their Rackspace Cloud account. We're only going to look at CloudFiles for this post.

**note** i'm going to use the tool, !(httpie)[https://github.com/jkbr/httpie] since its awesome and does some really nice formatting.  it works much like curl. 

Lets look at the users and roles available in identity:

    http -b -j get https://identity.api.rackspacecloud.com/v2.0/users x-auth-token:<auth token>
    {
        "users": [
            {
                "RAX-AUTH:defaultRegion": "DFW",
                "RAX-AUTH:domainId": "<tenant id>",
                "email": "<user email>",
                "enabled": true,
                "id": "<user id>",
                "username": "<user name>"
            }
        ]
    }

and:

    http -b -j get https://identity.api.rackspacecloud.com/v2.0/OS-KSADM/roles x-auth-token:<auth token>
    {
        "roles": [
            {
                "RAX-AUTH:Weight": "1000",
                "RAX-AUTH:propagate": "false",
                "description": "Object Store Admin Role for Account User",
                "id": "<id>",
                "name": "object-store:admin",
                "serviceId": "<service id>"
            },
            {
                "RAX-AUTH:Weight": "1000",
                "RAX-AUTH:propagate": "false",
                "description": "Object Store Observer Role for Account User",
                "id": "<id>",
                "name": "object-store:observer",
                "serviceId": "<service id>"
            },
            {
                "RAX-AUTH:Weight": "1000",
                "RAX-AUTH:propagate": "false",
                "description": "Full Access Admin Role for Account User",
                "id": "<id>",
                "name": "admin",
                "serviceId": "<service id>"
            },
            {
                "RAX-AUTH:Weight": "1000",
                "RAX-AUTH:propagate": "false",
                "description": "Read-Only Access Role for Account User",
                "id": "<id>",
                "name": "observer",
                "serviceId": "<service id>"
            }
        ]
    }

**note** i've removed the bulk of the roles and only showed the ones that matter for CloudFiles

if adding a user is what you're after, its pretty simple.  Our user called 'scott_test_user' with email of 'test_user@testdomain.com':

    http -j post https://identity.api.rackspacecloud.com/v2.0/users x-auth-token:9fd8ce103db942eb971b6b01b9697b64 user='{"username": "scott_test_user", "email": "test_user@testdomain.com", "enabled": true}'

    {
        "user": {
            "OS-KSADM:password": "<generated password>",
            "RAX-AUTH:defaultRegion": "ORD",
            "RAX-AUTH:domainId": "<domain id>",
            "email": "test_user@testdomain.com",
            "enabled": true,
            "id": "<user id>",
            "username": "scott_test_user"
        }
    }

**note** the username is going to need to be unique.  now you can't create one called scott_test_user.

Now for the really easy part, adding the user to your CloudFiles container.  There are two pieces of metadata that control the read and write access to a container, they are X-Container-Read and X-Container-Write.  Adding the user to the metadata will give access to the user. 

    http post https://storage101.iad3.clouddrive.com/v1/<storage tenant>/test_auth x-auth-token:<auth token> x-container-read:scott_test_user

the result should be a 204 code, which means it should have saved the metadata info.  if you do a HEAD on the container, the x-container-read metadata should be set:

    http head https://storage101.iad3.clouddrive.com/v1/<storage tenant>/test x-auth-token:<auth token>

    HTTP/1.1 204 No Content
    Accept-Ranges: bytes
    Content-Length: 0
    Content-Type: text/plain; charset=utf-8
    Date: Tue, 01 Oct 2013 18:45:38 GMT
    X-Container-Bytes-Used: 17
    X-Container-Object-Count: 2
    X-Container-Read: scott_test_user
    X-Timestamp: 1374263415.98445

The user can now login to auth with their password and get a token that allows them access to the CloudFiles container you gave them permission to.  Careful, setting _WRITE_ permission will allow them to delete objects. 

[User Documentation](http://docs.rackspace.com/auth/api/v2.0/auth-client-devguide/content/User_Calls.html)
[Roles Documentation](http://docs.rackspace.com/auth/api/v2.0/auth-client-devguide/content/Role_Calls.html)

