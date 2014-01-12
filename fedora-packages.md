---
title: Fedora Packages JSON API
---

<!-- vim: syn=markdown
-->

## API Notes

Fedora Packages provides a JSON API, but it's not documented. This document
provides some notes which may be useful to anyone ever looking to use the API.

### Accessing the API

#### General Information

* The global JSON endpoint is
  `https://apps.fedoraproject.org/packages/fcomm_connector/`

* The API works a bit oddly in that the JSON is actually part of the GET
  request, and goes at the end of the URL itself, after being URL encoded.

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
this Scala code will accomplish that:

* In all method calls except `contents`, you can specify `rows_per_page` and
  `start_row`, to let the API do pagination for you. The defaults are
  **10** for `rows_per_page` and **0** for `start_row`.

#### Endpoints

##### Search

The **search** endpoint is: `xapian/query/search_packages` so an example request
would be the properly URL-encoded form of this URL:

`https://apps.fedoraproject.org/packages/fcomm_connector/xapian/query/search_packages/{"filters": {"search": "chicken"}, "rows_per_page": 10, "start_row": 0}`

Parameters, other than `rows_per_page` and `start_now`:

* `filters` - a map containing a `"search"` key-value pair, whose value is the
  search query.

##### Releases

The **releases** endpoint is `bodhi/query/query_active_releases`.

Parameters, other than `rows_per_page` and `start_now`:

* `filters` - a map containing a `"package"` key-value pair, whose value is
  the package whose releases are being looked up.
