---
title: Fedora Packages JSON API
---

<!-- vim: syn=markdown
-->

## API Notes

Fedora Packages provides a JSON API, but it's not documented. This document
provides some notes which may be useful to anyone ever looking to use the API.

### Implementations

* [Haskell](https://github.com/codeblock/fedora-packages-hs/)
    ([hackage](http://hackage.haskell.org/package/fedora-packages))
* [Python](http://pkgwat.readthedocs.org/en/latest/) (reference)
* [Ruby](https://github.com/daviddavis/pkgwat)
* [Scala](https://github.com/CodeBlock/pkgwat-scala)

### Accessing the API

#### General Information

* The global JSON endpoint is
  `https://apps.fedoraproject.org/packages/fcomm_connector/`

* The API works a bit oddly in that the JSON is actually part of the GET
  request, and goes at the **end of the URL itself**, after being URL encoded.
    * There is an exception to this for the `yum/get_file_tree` endpoint, which
      takes its parameters as regular GET parameters.

For example, this fairly ugly-looking URL searches for packages with "chicken"
in the name, and returns JSON containing the result.

```json
$ http "https://apps.fedoraproject.org/packages/fcomm_connector/xapian/query/search_packages/%7B%22filters%22%3A%7B%22search%22%3A%22chicken%22%7D%2C%22rows_per_page%22%3A10%2C%22start_row%22%3A0%7D" | json_reformat
{
    "start_row": 0,
    "rows_per_page": 10,
    "rows": [
        {
            "summary": "A practical and portable Scheme system",
            "upstream_url": "http://call-cc.org",
            "icon": "package_128x128",
            "sub_pkgs": [
                {
                    "summary": "Documentation files for <span class=\"match\">CHICKEN</span> scheme.",
                    "icon": "package_128x128",
                    "link": "chicken-doc",
                    "description": "Documentation for <span class=\"match\">CHICKEN</span> (<span class=\"match\">chicken</span>-scheme).",
                    "name": "<span class=\"match\">chicken</span>-doc"
                }
            ],
            "name": "<span class=\"match\">chicken</span>",
            "devel_owner": "codeblock",
            "link": "chicken",
            "description": "<span class=\"match\">CHICKEN</span> is a compiler for the Scheme programming language.\n<span class=\"match\">CHICKEN</span> produces portable, efficient C, supports almost all of the R5RS\nScheme language standard, and includes many enhancements and extensions."
        }
    ],
    "total_rows": 1,
    "visible_rows": 1
}
```

* As you can see, the JSON returns some HTML in the response - so you'll want to
strip out those tags, before processing the data in most cases.
    * [upstream bug](https://github.com/fedora-infra/fedora-packages/issues/24)

* In all method calls except `yum/get_file_tree`, you can specify
  `rows_per_page` and `start_row`, to let the API do pagination for you. The
  defaults are **10** for `rows_per_page` and **0** for `start_row`.

* In all method calls except `yum/get_file_tree`, you MUST provide a `filters`
  key, with appropriate subfields.

* All method responses, with the exception of `yum/get_file_tree` are in the
  `rows` key. Other fields that are always present are `start_row`,
  `rows_per_page`, `total_rows`, and `visible_rows`.

#### Endpoints

Here is a breakdown of all available endpoints in the Fedora Packages API. Note
that we always exclude the `rows_per_page` and `start_row` fields. These are
valid in all endpoints except `yum/get_file_tree` which works very differently.

Also, all sample URLs we show should be URL encoded before being used. We keep
them in standard JSON form for ease of reading. For testing and experimenting,
the `httpie` program will automatically encode the URLs properly for you. Mind
your quote-escaping, however.

##### Search

The endpoint for **searching packages** is: `xapian/query/search_packages/`.

An **example request** looks like this:

`https://apps.fedoraproject.org/packages/fcomm_connector/xapian/query/search_packages/{"filters": {"search": "chicken"}, "rows_per_page": 10, "start_row": 0}`

###### `filters`

* `search` - the package name/description to search for.

###### Example Response

```json
{
    "rows": [
        {
            "description": "Python API for pkgwat",
            "devel_owner": "ralph",
            "icon": "package_128x128",
            "link": "python-pkgwat-api",
            "name": "python-pkgwat-api",
            "sub_pkgs": [
                {
                    "description": "Python API for pkgwat",
                    "icon": "package_128x128",
                    "link": "python3-python-pkgwat-api",
                    "name": "python3-python-pkgwat-api",
                    "summary": "Python API for querying the packages webapp"
        }],
            "summary": "Python API for querying the packages webapp",
            "upstream_url": "http://pypi.python.org/pypi/pkgwat.api"
    }],
    "rows_per_page": 10,
    "start_row": 0,
    "total_rows": 1,
    "visible_rows": 1
}
```

##### Releases

The endpoint for looking up **releases** is
`bodhi/query/query_active_releases/`.

###### `filters`

* `package` - the package to look up releases for.

###### Example Response

```json
{
    "rows": [
        {
            "release": "Rawhide",
            "stable_version": "3.4.3-27.fc18",
            "testing_version": "Not Applicable"
        },
        {
            "release": "Fedora 18",
            "stable_version": "3.4.3-27.fc18",
            "testing_version": "None"
        },
        {
            "release": "Fedora 17",
            "stable_version": "3.4.3-26.fc17",
            "testing_version": "None"
        },
        {
            "release": "Fedora 16",
            "stable_version": "3.4.3-25.fc15",
            "testing_version": "None"
        },
        {
            "release": "Fedora EPEL 6",
            "stable_version": "None",
            "testing_version": "None"
        },
        {
            "release": "Fedora EPEL 5",
            "stable_version": "3.4.3-12.el5.1",
            "testing_version": "None"
        }],
    "rows_per_page": 10,
    "start_row": 0,
    "total_rows": 6,
    "visible_rows": 6
}
```

##### Builds

The endpoint for looking up **builds** in Koji for a given package is
`koji/query/query_builds/`.

###### `filters`

* `package` - the package name
* `state` - A string that is either empty or a single digit, to filter
  build status
    * `""` - all build statuses
    * `"0"` - currently building
    * `"1"` - successfully built
    * `"2"` - deleted build
    * `"3"` - failed to build
    * `"4"` - cancelled build

Although this representation is unfortunate, this is a great place to employ sum-types in an implementation of the API.

###### Example Response

```json
 {
     "rows": [
         {
             "build_id": 332286,
             "completion_time": "2012-07-19 04:01:20.495063",
             "completion_time_display":
             {
                 "elapsed": "4 minutes",
                 "time": "04:01 AM UTC",
                 "when": "27 days ago"
             },
             "completion_ts": 1342670480.49506,
             "creation_event_id": 4896507,
             "creation_time": "2012-07-19 03:57:20.152193",
             "creation_ts": 1342670240.15219,
             "epoch": null,
             "name": "ccze",
             "nvr": "ccze-0.2.1-10.fc18",
             "owner_id": 131,
             "owner_name": "ausil",
             "package_id": 8962,
             "package_name": "ccze",
             "release": "10.fc18",
             "state": 1,
             "state_str": "complete",
             "task_id": 4250727,
             "version": "0.2.1",
             "volume_id": 0,
             "volume_name": "DEFAULT"
         },
         {
             "build_id": 298617,
             "completion_time": "2012-02-10 13:15:51.600556",
             "completion_time_display":
             {
                 "elapsed": "3 minutes",
                 "time": "01:15 PM UTC",
                 "when": "6 months ago"
             },
             "completion_ts": 1328879751.60056,
             "creation_event_id": 4407860,
             "creation_time": "2012-02-10 13:12:44.186579",
             "creation_ts": 1328879564.18658,
             "epoch": null,
             "name": "ccze",
             "nvr": "ccze-0.2.1-9.fc18",
             "owner_id": 1374,
             "owner_name": "ppisar",
             "package_id": 8962,
             "package_name": "ccze",
             "release": "9.fc18",
             "state": 1,
             "state_str": "complete",
             "task_id": 3778561,
             "version": "0.2.1",
             "volume_id": 0,
             "volume_name": "DEFAULT"
         }],
     "rows_per_page": 2,
     "start_row": 0,
     "total_rows": 15,
     "visible_rows": 2
 }
```

##### Updates

The endpoint for listing **updates** submitted to **Bodhi** is
`bodhi/query/query_updates/`.

###### `filters`

* `package` - the package name
* `release` - the Fedora (or EPEL) release to look for. In the format
  of `"f19"`, `"rawhide"`, and `"el6"`, but works for all releases
  known to Bodhi. The empty string (`""`) implies all releases.
* `status` - the Bodhi status of the filter by. One of:
    * `""` (empty string = all)
    * `"stable"`
    * `"testing"`
    * `"pending"`
    * `"obsolete"`

###### Example Response

```json
{
    "rows": [
    {
        "actions": "",
        "date_pushed": "10 Aug 2010",
        "date_pushed_display": "2 years ago",
        "date_submitted_display": "2 years ago",
        "details": "10 Aug 2010",
        "dist_updates": [
        {
            "approved": null,
            "bugs": [
                {
                    "bz_id": 578958,
                    "parent": False,
                    "security": False,
                    "title": "[abrt] crash in ccze-0.2.1-5.fc12: Process \
/usr/bin/ccze was killed by signal 11"
                },
                {
                    "bz_id": 612866,
                    "parent": False,
                    "security": False,
                    "title": "[abrt] crash in ccze-0.2.1-5.fc12: \
parse_opt: Process /usr/bin/ccze was killed by signal 11 (SIGSEGV)"
                }],
            "builds": [
            {
                "nvr": "ccze-0.2.1-6.el5",
                "package":
                {
                    "committers": ["hubbitus"],
                    "name": "ccze",
                    "suggest_reboot": False
                }
            }],
            "close_bugs": True,
            "comments": [
                {
                    "anonymous": False,
                    "author": "bodhi",
                    "group": null,
                    "karma": 0,
                    "text": "This update has been submitted for \
testing by hubbitus. ",
                    "timestamp": "2010-08-10 07:30:07",
                    "update_title": "ccze-0.2.1-6.el5"
                },
                {
                    "anonymous": False,
                    "author": "bodhi",
                    "group": null,
                    "karma": 0,
                    "text": "This update has been pushed to testing",
                    "timestamp": "2010-08-10 17:55:25",
                    "update_title": "ccze-0.2.1-6.el5"
                },
                {
                    "anonymous": False,
                    "author": "bodhi",
                    "group": null,
                    "karma": 0,
                    "text": "This update has reached 14 days in \
testing and can be pushed to stable now if the maintainer wishes",
                    "timestamp": "2010-08-24 23:06:30",
                    "update_title": "ccze-0.2.1-6.el5"
                },
                {
                    "anonymous": False,
                    "author": "bodhi",
                    "group": null,
                    "karma": 0,
                    "text": "This update has been submitted for \
stable by hubbitus. ",
                    "timestamp": "2010-08-25 08:00:11",
                    "update_title": "ccze-0.2.1-6.el5"
                },
                {
                    "anonymous": False,
                    "author": "bodhi",
                    "group": null,
                    "karma": 0,
                    "text": "This update has been pushed to stable",
                    "timestamp": "2010-08-25 18:54:46",
                    "update_title": "ccze-0.2.1-6.el5"
                }],
            "critpath": False,
            "critpath_approved": True,
            "date_modified": null,
            "date_pushed": "2010-08-10 17:23:39",
            "date_submitted": "2010-08-10 07:29:56",
            "karma": 0,
            "nagged": null,
            "notes": "",
            "release":
            {
                "dist_tag": "dist-5E-epel",
                "id_prefix": "FEDORA-EPEL",
                "locked": False,
                "long_name": "Fedora EPEL 5",
                "name": "EL-5"
            },
            "release_name": "Fedora EPEL 5",
            "request": null,
            "stable_karma": 3,
            "status": "stable",
            "submitter": "hubbitus",
            "title": "ccze-0.2.1-6.el5",
            "type": "bugfix",
            "unstable_karma": -3,
            "updateid": "FEDORA-EPEL-2010-3205",
            "version": "0.2.1-6.el5"
        }],
        "id": "ccze-0.2.1-6.el5",
        "karma_level": "meh",
        "karma_str": " 0",
        "name": "ccze",
        "nvr": "ccze-0.2.1-6.el5",
        "package_name": "ccze",
        "releases": ["Fedora EPEL 5"],
        "request_id": "ccze021-6el5",
        "status": "stable",
        "title": "ccze-0.2.1-6.el5",
        "versions": ["0.2.1-6.el5"]
    }],
    "rows_per_page": 10,
    "start_row": 0,
    "total_rows": 2,
    "visible_rows": 1
}
```

