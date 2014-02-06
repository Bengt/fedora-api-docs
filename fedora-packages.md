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
$ http 'https://apps.fedoraproject.org/packages/fcomm_connector/xapian/query/search_packages/%7B%22filters%22%3A%7B%22search%22%3A%22chicken%22%7D%2C%22rows_per_page%22%3A10%2C%22start_row%22%3A0%7D' | json_reformat
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
    u 'rows': [
        {
            u 'description': u 'Python API for pkgwat',
            u 'devel_owner': u 'ralph',
            u 'icon': u 'package_128x128',
            u 'link': u 'python-pkgwat-api',
            u 'name': u 'python-pkgwat-api',
            u 'sub_pkgs': [
                {
                    u 'description': u 'Python API for pkgwat',
                    u 'icon': u 'package_128x128',
                    u 'link': u 'python3-python-pkgwat-api',
                    u 'name': u 'python3-python-pkgwat-api',
                    u 'summary': u 'Python API for querying the packages webapp'
        }],
            u 'summary': u 'Python API for querying the packages webapp',
            u 'upstream_url': u 'http://pypi.python.org/pypi/pkgwat.api'
    }],
    u 'rows_per_page': 10,
    u 'start_row': 0,
    u 'total_rows': 1,
    u 'visible_rows': 1
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
    u 'rows': [
        {
            u 'release': u 'Rawhide',
            u 'stable_version': u '3.4.3-27.fc18',
            u 'testing_version': u 'Not Applicable'
        },
        {
            u 'release': u 'Fedora 18',
            u 'stable_version': u '3.4.3-27.fc18',
            u 'testing_version': u 'None'
        },
        {
            u 'release': u 'Fedora 17',
            u 'stable_version': u '3.4.3-26.fc17',
            u 'testing_version': u 'None'
        },
        {
            u 'release': u 'Fedora 16',
            u 'stable_version': u '3.4.3-25.fc15',
            u 'testing_version': u 'None'
        },
        {
            u 'release': u 'Fedora EPEL 6',
            u 'stable_version': u 'None',
            u 'testing_version': u 'None'
        },
        {
            u 'release': u 'Fedora EPEL 5',
            u 'stable_version': u '3.4.3-12.el5.1',
            u 'testing_version': u 'None'
        }],
    u 'rows_per_page': 10,
    u 'start_row': 0,
    u 'total_rows': 6,
    u 'visible_rows': 6
}
```

##### Builds

The endpoint for looking up **builds** in Koji for a given package is
`koji/query/query_builds/`.

###### `filters`

* `package` - the package name
* `state` - A string that is either empty or a single digit, to filter
  build status
    * `''` - all build statuses
    * `'0'` - currently building
    * `'1'` - successfully built
    * `'2'` - deleted build
    * `'3'` - failed to build
    * `'4'` - cancelled build

Although this representation is unfortunate, this is a great place to employ sum-types in an implementation of the API.

###### Example Response

```json
 {
     u 'rows': [
         {
             u 'build_id': 332286,
             u 'completion_time': u '2012-07-19 04:01:20.495063',
             u 'completion_time_display':
             {
                 u 'elapsed': u '4 minutes',
                 u 'time': u '04:01 AM UTC',
                 u 'when': u '27 days ago'
             },
             u 'completion_ts': 1342670480.49506,
             u 'creation_event_id': 4896507,
             u 'creation_time': u '2012-07-19 03:57:20.152193',
             u 'creation_ts': 1342670240.15219,
             u 'epoch': None,
             u 'name': u 'ccze',
             u 'nvr': u 'ccze-0.2.1-10.fc18',
             u 'owner_id': 131,
             u 'owner_name': u 'ausil',
             u 'package_id': 8962,
             u 'package_name': u 'ccze',
             u 'release': u '10.fc18',
             u 'state': 1,
             u 'state_str': u 'complete',
             u 'task_id': 4250727,
             u 'version': u '0.2.1',
             u 'volume_id': 0,
             u 'volume_name': u 'DEFAULT'
         },
         {
             u 'build_id': 298617,
             u 'completion_time': u '2012-02-10 13:15:51.600556',
             u 'completion_time_display':
             {
                 u 'elapsed': u '3 minutes',
                 u 'time': u '01:15 PM UTC',
                 u 'when': u '6 months ago'
             },
             u 'completion_ts': 1328879751.60056,
             u 'creation_event_id': 4407860,
             u 'creation_time': u '2012-02-10 13:12:44.186579',
             u 'creation_ts': 1328879564.18658,
             u 'epoch': None,
             u 'name': u 'ccze',
             u 'nvr': u 'ccze-0.2.1-9.fc18',
             u 'owner_id': 1374,
             u 'owner_name': u 'ppisar',
             u 'package_id': 8962,
             u 'package_name': u 'ccze',
             u 'release': u '9.fc18',
             u 'state': 1,
             u 'state_str': u 'complete',
             u 'task_id': 3778561,
             u 'version': u '0.2.1',
             u 'volume_id': 0,
             u 'volume_name': u 'DEFAULT'
         }],
     u 'rows_per_page': 2,
     u 'start_row': 0,
     u 'total_rows': 15,
     u 'visible_rows': 2
 }
```

##### Updates

The endpoint for listing **updates** submitted to **Bodhi** is
`bodhi/query/query_updates/`.

###### `filters`

* `package` - the package name
* `release` - the Fedora (or EPEL) release to look for. In the format
  of `'f19'`, `'rawhide'`, and `'el6'`, but works for all releases
  known to Bodhi. The empty string (`''`) implies all releases.
* `status` - the Bodhi status of the filter by. One of:
    * `''` (empty string = all)
    * `'stable'`
    * `'testing'`
    * `'pending'`
    * `'obsolete'`

###### Example Response

```json
{
    u 'rows': [
    {
        u 'actions': '',
        u 'date_pushed': u '10 Aug 2010',
        u 'date_pushed_display': u '2 years ago',
        u 'date_submitted_display': u '2 years ago',
        u 'details': u '10 Aug 2010',
        u 'dist_updates': [
        {
            u 'approved': None,
            u 'bugs': [
                {
                    u 'bz_id': 578958,
                    u 'parent': False,
                    u 'security': False,
                    u 'title': u '[abrt] crash in ccze-0.2.1-5.fc12: Process \
/usr/bin/ccze was killed by signal 11'
                },
                {
                    u 'bz_id': 612866,
                    u 'parent': False,
                    u 'security': False,
                    u 'title': u '[abrt] crash in ccze-0.2.1-5.fc12: \
parse_opt: Process /usr/bin/ccze was killed by signal 11 (SIGSEGV)'
                }],
            u 'builds': [
            {
                u 'nvr': u 'ccze-0.2.1-6.el5',
                u 'package':
                {
                    u 'committers': [u 'hubbitus'],
                    u 'name': u 'ccze',
                    u 'suggest_reboot': False
                }
            }],
            u 'close_bugs': True,
            u 'comments': [
                {
                    u 'anonymous': False,
                    u 'author': u 'bodhi',
                    u 'group': None,
                    u 'karma': 0,
                    u 'text': u 'This update has been submitted for \
testing by hubbitus. ',
                    u 'timestamp': u '2010-08-10 07:30:07',
                    u 'update_title': u 'ccze-0.2.1-6.el5'
                },
                {
                    u 'anonymous': False,
                    u 'author': u 'bodhi',
                    u 'group': None,
                    u 'karma': 0,
                    u 'text': u 'This update has been pushed to testing',
                    u 'timestamp': u '2010-08-10 17:55:25',
                    u 'update_title': u 'ccze-0.2.1-6.el5'
                },
                {
                    u 'anonymous': False,
                    u 'author': u 'bodhi',
                    u 'group': None,
                    u 'karma': 0,
                    u 'text': u 'This update has reached 14 days in \
testing and can be pushed to stable now if the maintainer wishes',
                    u 'timestamp': u '2010-08-24 23:06:30',
                    u 'update_title': u 'ccze-0.2.1-6.el5'
                },
                {
                    u 'anonymous': False,
                    u 'author': u 'bodhi',
                    u 'group': None,
                    u 'karma': 0,
                    u 'text': u 'This update has been submitted for \
stable by hubbitus. ',
                    u 'timestamp': u '2010-08-25 08:00:11',
                    u 'update_title': u 'ccze-0.2.1-6.el5'
                },
                {
                    u 'anonymous': False,
                    u 'author': u 'bodhi',
                    u 'group': None,
                    u 'karma': 0,
                    u 'text': u 'This update has been pushed to stable',
                    u 'timestamp': u '2010-08-25 18:54:46',
                    u 'update_title': u 'ccze-0.2.1-6.el5'
                }],
            u 'critpath': False,
            u 'critpath_approved': True,
            u 'date_modified': None,
            u 'date_pushed': u '2010-08-10 17:23:39',
            u 'date_submitted': u '2010-08-10 07:29:56',
            u 'karma': 0,
            u 'nagged': None,
            u 'notes': '',
            u 'release':
            {
                u 'dist_tag': u 'dist-5E-epel',
                u 'id_prefix': u 'FEDORA-EPEL',
                u 'locked': False,
                u 'long_name': u 'Fedora EPEL 5',
                u 'name': u 'EL-5'
            },
            u 'release_name': u 'Fedora EPEL 5',
            u 'request': None,
            u 'stable_karma': 3,
            u 'status': u 'stable',
            u 'submitter': u 'hubbitus',
            u 'title': u 'ccze-0.2.1-6.el5',
            u 'type': u 'bugfix',
            u 'unstable_karma': -3,
            u 'updateid': u 'FEDORA-EPEL-2010-3205',
            u 'version': u '0.2.1-6.el5'
        }],
        u 'id': u 'ccze-0.2.1-6.el5',
        u 'karma_level': u 'meh',
        u 'karma_str': u ' 0',
        u 'name': u 'ccze',
        u 'nvr': u 'ccze-0.2.1-6.el5',
        u 'package_name': u 'ccze',
        u 'releases': [u 'Fedora EPEL 5'],
        u 'request_id': u 'ccze021-6el5',
        u 'status': u 'stable',
        u 'title': u 'ccze-0.2.1-6.el5',
        u 'versions': [u '0.2.1-6.el5']
    }],
    u 'rows_per_page': 10,
    u 'start_row': 0,
    u 'total_rows': 2,
    u 'visible_rows': 1
}
```

