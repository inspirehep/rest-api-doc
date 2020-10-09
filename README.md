# INSPIRE REST API

This document explains how to access [INSPIRE](https://inspirehep.net) metadata programmatically through a REST API.

- [INSPIRE REST API](#inspire-rest-api)
  * [Questions & comments](#questions-and-comments)
  * [Terms of use](#terms-of-use)
  * [API Overview](#api-overview)
  * [Rate limiting](#rate-limiting)
  * [Obtaining a record](#obtaining-a-record)
    + [Internal identifiers](#internal-identifiers)
    + [External identifiers](#external-identifiers)
  * [Single-record response](#single-record-response)
    + [Links](#links)
    + [Metadata](#metadata)
  * [Changing formats](#changing-formats)
  * [Searching](#searching)
    + [Search query](#search-query)
    + [Sort order](#sort-order)
    + [Pagination](#pagination)
    + [Search response](#search-response)
    + [Metadata filtering](#metadata-filtering)
  * [API usage in the wild](#api-usage-in-the-wild)


## Questions and comments

If you have any issues using the API, would like some help, or have some suggestions for improving the API or its documentation, please open [an issue](https://github.com/inspirehep/rest-api-doc/issues) or [send us an email](mailto:feedback@inspirehep.net).

## Terms of use

Usage of the API is governed by our [terms of use](https://inspirehep.net/help/knowledge-base/terms-of-use/). As explained there in more detail, most of the metadata is available under a [CC0 license](https://creativecommons.org/publicdomain/zero/1.0/), but restrictions apply to some fields, and bulk collection of email addresses is not allowed.

## API Overview

The API is generally RESTful and returns results in JSON by default.
This means for example that it will return a 404 HTTP status code if a record can't be found.

In general, most pages you get through the website have a corresponding representation in the API obtained by prefixing the path component of the URL with `/api/`.
For example, the data displayed at
```
https://inspirehep.net/literature?sort=mostrecent&size=25&page=1&q=title api
```
is available through the API at
```
https://inspirehep.net/api/literature?sort=mostrecent&size=25&page=1&q=title api
```

Currently only read-only operations are allowed and they all use the `GET` HTTP method.

Note that all examples are displayed in a human-readable way, but the query parameters often need to be [URL-encoded](https://en.wikipedia.org/wiki/Percent-encoding). In particular, spaces need to be replaced by `%20`.

## Rate limiting

In order to avoid overwhelming the server, we enforce rate limits per IP address: every IP address is allowed 50 requests, then at most 2 requests per second. If you exceed those limits, you will receive a response with HTTP status code 429 and a `x-retry-in` header telling you how long to wait before retrying.

## Obtaining a record

To get the metadata in a single record, use the following type of URL:
```
https://inspirehep.net/api/{identifier-type}/{identifier-value}
```

Two main categories of record identifiers (that is pairs of `{identifier-type}` and `{identifier-value}`) are supported.

### Internal identifiers

These are the same identifiers as appear in the URLs on the website and can also be used for [searching](#searching). The `{identifier-type}` can take the following values:

* `literature`
* `authors`
* `conferences`
* `seminars`
* `journals`
* `jobs`
* `experiments`
* `data`

and the `{identifier-value}` is a number identifying the given record in the INSPIRE database (also called record ID or `recid`). For example,
```
https://inspirehep.net/api/literature/451647
```
is the record of Maldacena's famous [AdS/CFT paper](https://inspirehep.net/literature/451647), whereas
```
https://inspirehep.net/api/conferences/1642486
```
is the record of the [ICHEP 2018 conference](https://inspirehep.net/conferences/1642486).

### External identifiers

These are [persistent identifiers](https://en.wikipedia.org/wiki/Persistent_identifier) that were not assigned by INSPIRE but nevertheless uniquely identify a record in INSPIRE (if a record for the corresponding identifier exists in the system).

The following external identifiers can be used:

|`{identifier-type}`|`{identifier-value}` (examples)|Usage|
|---|---|---|
|`doi`|`10.1103/PhysRevLett.19.1264`|to get a literature record given a [DOI](https://www.doi.org)|
|`arxiv`|`1207.7214`, `hep-ph/0603175`|to get a literature record given an [arXiv](https://arxiv.org) identifier|
|`orcid`|`0000-0003-3897-046X`|to get an author record given an [ORCID](https://orcid.org) iD|

## Single-record response

By default, the API response when retrieving a single record will be in the JSON format and contain the following keys:

|Key|Description|
|---|---|
|`id`|Identifier used to retrieve the record|
|`created`|Creation timestamp of the record in UTC|
|`updated`|Last update timestamp of the record in UTC|
|`links`|Links to resources related to the record|
|`metadata`|Metadata of the record|

Whatever identifier is used to retrieve the record, it will also be present (as well as the other identifiers belonging to this record) inside `metadata`.

### Links

The `links` object contains links to metadata related to this record but not directly included in the record (e.g. citation information), and [alternative serialization formats](#changing-formats) (e.g. BibTeX).

### Metadata

The `metadata` object contains the metadata of the record proper. All records have a `$schema` key, which links to a [JSON schema](https://json-schema.org/understanding-json-schema/index.html) (draft 4) that the record metadata obeys. Detailed documentation about the possible fields for each schema and their meaning can be found in the [schema documentation](https://inspire-schemas.readthedocs.io/en/latest/schemas/).

For example, the `metadata` of `Literature` records conforms to the `hep` schema, whose fields are documented [here](https://inspire-schemas.readthedocs.io/en/latest/schemas/hep.html).

## Changing formats

It is possible to obtain a representation of a record (or several records) in a different format from the default JSON. This can be done in two alternative ways:

* using the `format={format-name}` URL query string, or
* through [content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation), by setting the `Accept` [HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) to a specific MIME type.

Currently, the following formats are supported (only for `Literature` records):

|`{format-name}`|MIME type|Description|
|---|---|---|
|json|application/json|The default JSON format|
|bibtex|application/x-bibtex|The [BibTeX](https://en.wikipedia.org/wiki/BibTeX) citation format|
|latex-eu|application/vnd+inspire.latex.eu+x-latex|The LaTeX (EU) citation format|
|latex-us|application/vnd+inspire.latex.us+x-latex|The LaTeX (US) citation format|

Links to alternative formats can also be found in the `links` object inside the JSON response.

For example, to obtain [Glashow's famous paper on weak interactions](https://inspirehep.net/literature/4328) in BibTeX format, use either a format parameter:
```
https://inspirehep.net/api/literature/4328?format=bibtex
```
or equivalently, content negotiation (the example uses the [`curl`](https://curl.haxx.se/) command line tool to set the header):
```
curl -H "Accept: application/x-bibtex" https://inspirehep.net/api/literature/4328
```

## Searching

In order to get results for a search rather than getting the data of a single record by its identifier, use a base URL of the following form:

```
https://inspirehep.net/api/{record-type}?{query-string}
```

The `{record-type}` must be one of:

* `literature`
* `authors`
* `conferences`
* `seminars`
* `journals`
* `jobs`
* `experiments`
* `data`

Note that these are the same as the [internal identifier types](#internal-identifiers).

The `{query-string}` may contain several `{parameter}={value}` pairs separated by `&`. The following parameters are always supported:

|`{parameter}`|Description of `{value}`|
|---|---|
|`q`|The search query|
|`sort`|The sort order|
|`size`|The number of results returned per page|
|`page`|The page number|
|`fields`|The fields in the metadata to be returned|

Additionally, depending on the `{record-type}`, different facet filters are available to restrict the set of results. They work exactly the same way as on the website.

For example, to obtain the 6th to 10th upcoming conferences, the following URL can be used:
```
https://inspirehep.net/api/seminars?size=5&page=2&start_date=upcoming
```
To obtain the 10 most recent papers cited at least 1000 times, use:
```
https://inspirehep.net/literature?sort=mostrecent&size=10&q=topcite 1000+
```

### Search query

The `q` query string argument allows to specify a search query that only matches a subset of records.

* For Literature records (obtained through the `/api/literature` endpoint), a custom search syntax is used for backwards compatibility with SPIRES and the old INSPIRE. It is explained [here](https://inspirehep.net/help/knowledge-base/inspire-paper-search/). Additionally, any field of the record metadata can be searched using its path given by concatenating the nested keys with `.`, followed by a `:` and the value to search for.

  For example, to find all papers having an abstract from Springer, the following search can be used:
```
https://inspirehep.net/api/literature?q=abstracts.source:Springer
```
  To find all conferences papers citing Edward Witten, you can use:
```
https://inspirehep.net/api/literature?q=tc conference paper and refersto a E.Witten.1
```
  To check whether a field exists, you can use a `*` wildcard. For example, to find all papers having a DOI, you can use:
```
https://inspirehep.net/api/literature?q=dois.value:*
```

* For other types of records, the [ElasticSearch query string](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-query-string-query.html#query-string-syntax) syntax is used.

  For example, to find all experiments using the CERN Proton-Synchotron (PS) accelerator, use
```
https://inspirehep.net/api/experiments?q=accelerator.value:PS
```

### Sort order

The order in which search results are returned depends on whether a search query is provided.

By default,
* if no search query is provided (no `q` query parameter), results are sorted with the most recent records first,
* if a search query is provided (through a `q` query parameter), results are sorted with the most relevant first.

This behavior can be overridden with the `sort={sort-order}` query parameter. The following options are supported:

|`{record-type}`|`{sort-order}`|Description|
|---|---|---|
|`literature`|`mostrecent`|Most recent records appear first (based on earliest date in metadata)|
|`literature`|`mostcited`|Records with most citations appear first|
|`jobs`|`mostrecent`|Most recently created jobs appear first|
|`jobs`|`deadline`|Jobs with the earliest deadline appear first|
|`conferences`|`dateasc`|Conferences with the earliest starting date appear first|
|`conferences`|`datedesc`|Conferences with the most recent starting date appear first|
|`seminars`|`dateasc`|Seminars with the earliest starting time appear first|
|`seminars`|`datedesc`|Seminars with the most recent starting time appear first|

For example, the following URL will return Edward Witten's 10 most cited papers:
```
https://inspirehep.net/api/literature?sort=mostcited&size=10&q=a E.Witten.1
```

### Pagination

Search results are returned in pages to limit the size of the response. By default, 10 results are returned per page and the first page of results is returned.
To get to the next pages, you can pass the page number to the `page` query parameter.

For example, use the following URL to get Edward Witten's 31st to 40th most cited papers.
```
https://inspirehep.net/api/literature?sort=mostcited&page=3&q=a E.Witten.1
```

To go the next page, the `next` URL of the `links` object in the response can be followed (when using the default JSON format).

The number of results per page can be overriden with the `size` query parameter. In order not to overload the server, the maximum allowed value is `1000`, and you'll get a response with HTTP status code 400 if you exceed it.

For example, to get the 50 most cited papers of Edward Witten at once, the following URL can be used:
```
https://inspirehep.net/api/literature?sort=mostcited&size=50&q=a E.Witten.1
```

### Search response

The response for a search is a JSON object with the following keys:

* `hits`: contains the total number of results in `total` and the records in `hits` (which is an array whose elements have the same structure as in the [single-record response](#single-record-response))
* `links`: links to related resources, such as alternative serializations of the search results and the next page in `next`.

Note that the record metadata (in `hits.hits.metadata`) contains more fields than in the [single-record response](#single-record-response). Most of those are for internal use: **any field not part of the [schema](https://inspire-schemas.readthedocs.io/en/latest/schemas/) should not be relied on, and there is no guarantee that it will remain present or that its content won't change**, with the exception of:

* for `/api/literature`

|key|value (example)|description|
|---|---|---|
|`earliest_date`|`2020-03-18`|the earliest date on the record|
|`citation_count`|243|the total number of citations received by this record|
|`citation_count_without_self_citations`|213|the number of citations received by this record, excluding [self-citations](https://inspirehep.net/help/knowledge-base/citation-metrics/)|

### Metadata filtering

Sometimes, you might be interested in only some specific fields of the record metadata and not in the whole record. To avoid generating and downloading the full response, which can be quite large, the `fields` query parameter can be used. It should be set to a comma-separated list of fields that need to be present in the record metadata.

For example, the following URL will return only the titles, author names and links to affiliation records of papers with more than 1000 citations:
```
https://inspirehep.net/literature?fields=titles,authors.full_name,authors.affiliations.record&q=topcite 1000+
```

Note that it is not possible to put limits on the number of elements of an array, but only select whether that array should appear at all. For example, it's not possible to select only the first 10 authors, but it's possible to avoid returning authors by not putting `authors` (or any of its subfields) among the `fields`.

## API usage in the wild

Several tools in different languages are using this API. Their code might serve as a useful source of real-world examples.

* PHP:
  + [INSPIREJSON](https://github.com/csanadm/inspirejson)
* Python:
  + [citations.py](https://github.com/efranzin/python/blob/master/citations.py)
* Objective-C:
  + [spires.app/inspire.app](https://github.com/yujitach/inspire)
  + [BibDesk](https://sourceforge.net/projects/bibdesk/)

If you would like your project to be listed, don't hesitate to [let us know](#questions-and-comments).
