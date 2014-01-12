---
title: Fedora Copr JSON API
---

<!-- vim: syn=markdown
-->

## API Notes

There is some documentation on the
[copr site](http://copr.fedoraproject.org/api), but I have found it to be
lacking examples and also a bit out of date.

### Accessing the API

#### General Information

* The global JSON endpoint is
  `http://copr.fedoraproject.org/api/coprs/`

* Several endpoints require authentication (which uses
  [basic access authentication](https://en.wikipedia.org/wiki/Basic_access_authentication)).
  However, you can send authenticated requests to **all** endpoints, and those
  which do not require it will simply ignore it.
    * This makes it easier to go about writing API clients without having to
      handle a lot of special cases.

* All endpoints require a trailing slash (`/`) on them. Otherwise, Copr will
  generate a redirect which must be handled by your HTTP client (and also
  wastes bandwidth which slows things down).

* The API seems to disregard `Accept` headers. Sending `Accept: text/html`
  yields `Content-Type: application/json` regardless. That said, you should
  probably send the proper `Accept: application/json` header anyway, in case
  this functionality changes in the future.

* A malformed request usually results in a **500** error being thrown by Copr
  in addition to this being in the response: `"output": "notok"`. Occasionally,
  there will be an `error` key with useful information, although this seems to
  be rare (the `error` value seems to usually be empty, or too general to be of
  use).

* You may see references around to "copr-fe.cloud.fedoraproject.org" which was
  the location of copr while it was being created. This now functions as an
  alias for "copr.fedoraproject.org" which is preferred.

#### Authentication

As noted above, even endpoints that don't require authentication will function
just fine, if given authentication credentials. It will simply drop them.

Authentication is performed using
[basic access authentication](https://en.wikipedia.org/wiki/Basic_access_authentication)
which should be available in nearly any HTTP library.

To obtain credentials, you must:

- [Log in](http://copr.fedoraproject.org/login) to Copr.
- Navigate to the [API](http://copr.fedoraproject.org/api) page.
- Click the **Generate a new token** button.

You will see a block on the page that looks like this:

```
[copr-cli]
login = abcde
username = janedoe
token = 12345
copr_url = http://copr.fedoraproject.org
# expiration date: 2014-07-10
```

Take note of the **login** and **token** fields. The **login** field becomes
the "username" of the HTTP Basic Authentication transaction, and the **token**
becomes the "password".

Using httpie, an authenticated request should look something like this, given
the above example:

`http --auth abcde:12345 GET http://copr.fedoraproject.org/api/coprs/janedoe/`

#### Endpoints

There are currently endpoints to:

- List someone's copr projects
- Create a new copr project\*
- Add a new build\*
- Query a build's status\*

`*` denotes required authentication.

##### List someone's copr projects

`GET /api/coprs/[username]/`

This (unauthenticated) endpoint lists all copr projects belonging to
`[username]`.

A valid, example request is this:

`http GET http://coprfedoraproject.org/api/coprs/codeblock/`

...which yields a response in the form of:

```json
{
    "output": "ok", 
    "repos": [
        {
            "additional_repos": "", 
            "description": "Repository for unofficially packaged http://eval.so/ dependencies. It is my intention that these will one day become official Fedora packages. Most packages here are programming related.", 
            "instructions": "See individual repos at https://github.com/eval-so/ for instructions, or http://eval.so/api for instructions on using the hosted API.", 
            "name": "evalso", 
            "yum_repos": {
                "fedora-20-x86_64": "http://copr-be.cloud.fedoraproject.org/results/codeblock/evalso/fedora-20-x86_64/"
            }
        }
    ]
}
```

##### Create a new copr project\*

`POST /api/coprs/[username]/new/`

This endpoint creates a new copr project belonging to `[username]`. It requires
a `name` and at least one chroot.

Chroots are selected by listing them as **actual JSON fields** in the request,
with the value `y`.

A valid, example request is this:

`http  --auth login:token POST http://copr.fedoraproject.org/api/coprs/codeblock/new/ name=testproject fedora-20-x86_64=y fedora-rawhide-i386=y`

...which yields a response in the form of:

```json
{
    "message": "New project was successfully created.", 
    "output": "ok"
}
```

If a project with the given name already exists for the given user, the API
will return **500**, with this JSON body:

```json
{
    "error": "You already have project named 'testproject'.", 
    "output": "notok"
}
```

##### Add a new build\*

`POST /api/coprs/[username]/[copr_name]/`

This endpoint adds a build to an existing copr project.

It has 3 possible fields (bolded fields are required):

- **`pkgs`** - A space-separated list of URLs, which are .src.rpms, downloadable
  for building.
- `memory` - How much memory should be alloted to the build.
    - This ([appears](https://git.fedorahosted.org/cgit/copr.git/tree/coprs_frontend/coprs/constants.py)
      as though it) should be supplied in MB, with an upper bound of 4096 and a
      lower bound of 2048. Default is 2048.
- `timeout` - Timeout for the build (units?)
    - This ([appears](https://git.fedorahosted.org/cgit/copr.git/tree/coprs_frontend/coprs/constants.py) 
      as though it) should be supplied in seconds with a lower bound of 1 and
      an upper bound of 36000 (i.e., 10 hours). It appears as though 0 disables
      this feature (?).

According to [this file](https://git.fedorahosted.org/cgit/copr.git/tree/coprs_frontend/coprs/models.py#n64)
in the source code, only "proven" users can set `memory` and `timeout`, but it
is unclear how to determine whether or not a given user is "proven" or how to
obtain that status.

A valid, example request is this:

`http  --auth login:token POST http://copr.fedoraproject.org/api/coprs/codeblock/testproject/new_build/ pkgs='http://kojipkgs.fedoraproject.org//packages/dmtcp/2.1/1.fc20/src/dmtcp-2.1-1.fc20.src.rpm'`

...which yields a response in the form of:

```json
{
    "id": 1016, 
    "message": "Build was added to testproject.", 
    "output": "ok"
}
```

##### Query a build's status

`GET /api/coprs/build_status/[build_id]/`

This endpoint provides information about the current status of the given build.

A valid, example request is this:

`http --auth login:token GET http://copr.fedoraproject.org/api/coprs/build_status/1017/`

...which yields a response in the form of:

```json
{
    "output": "ok", 
    "status": "running"
}
```

The `status` field [will be](https://git.fedorahosted.org/cgit/copr.git/tree/coprs_frontend/coprs/models.py#n262)
one of "failed", "running", "pending", or "succeeded".
