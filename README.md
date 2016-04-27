
# Super API

Super API will make you super 'appy.

## Overview

Super API is essentially a JSON API specification like [JSON:API](https://jsonapi.org) or [HAL](https://stateless.co/hal_specification.html).  We believe it has advantages over both though (otherwise we wouldn't have bothered).

JSON:API is billed as an "anti-bikeshedding tool", yet leaves quite a lot of detail up to the implementer.  Super API is quite opinionated in an effort to leave less room for interpretation, although it borrows heavily from the former.

### Basic example

We'll use the canonical example of a blog system to whet your appetite.  Below is the sample response to a request for a list of blog posts.

```json
{
  "links": {
    "$next": "https://blog/api/posts?page[number]=2"
  },
  "elements": [
    {
      "links": {
        "$self": "https://blog/api/posts/1",
        "author": "https://blog/api/users/1",
        "comments": "https://blog/api/posts/1/comments"
      },
      "attributes": {
        "title": "Hello, world!",
        "content": "This is a test post."
      }
    },
    {
      "links": {
        "$self": "https://blog/api/posts/2",
        "author": "https://blog/api/users/1",
        "comments": "https://blog/api/posts/2/comments"
      },
      "attributes": {
        "title": "Test post 2",
        "content": "This is second test post."
      }
    }
  ],
  "meta": {
    "count": 8
  },
  "includes": [
    {
      "links": {
        "$self": "https://blog/api/users/1"
      },
      "attributes": {
        "name": "Fred Flintstone",
        "email": "fred@example.com"
      }
    },
    {
      "links": {
        "$self": "https://blog/api/posts/1/comments"
      },
      "elements": [
        {
          "links": {
            "$self": "https://blog/api/comments/1"
          },
          "attributes": {
            "content": "This is a comment!"
          }
        }
      ]
    },
    {
      "links": {
        "$self": "https://blog/api/posts/2/comments"
      },
      "elements": []
    }
  ]
}
```

This example demonstrates embedding multiple resources (including collections of resources) into the response body.  See below for a more thorough description.


## The Resource object

The *resource* object is the basic building block.  A resource object wrapping the normal old-fashioned JavaScript object `{name: 'Fred Flintstone', email: 'fred@example.com'}` looks like so:

```json
{
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  }
}
```

In words, it's just the object placed into a field called `attributes` in a wrapper object.  It might also have a `links` field, defining links to other related information:

```json
{
  "links": {
    "posts": "https://blog/api/posts?filter[authorId]=1"
  },
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  }
}
```

The special link name `$self` is used to give the canonical URL of the resource if it is served from a non-canonical URL.  For example, the two URLs `https://blog/api/users/1` and `http://blog/api/posts/1/author` might refer to the same resource, and the latter should include the `$self` link to state the canonical URL as the first one.

And it might have a `meta` field, which can have any implementation specific data.  It's quite a good place to put item counts and other such stuff:

```json
{
  "links": {
    "posts": "https://blog/api/posts?filter[authorId]=1"
  },
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  },
  "meta": {
    "postCount": 10
  }
}
```

How about collections of resources?  These are given in the `elements` field of the resource object.

```json
{
  "elements": [
    {
      "links": {
        "$self": "https://blog/api/users/1"
      },
      "attributes": {
        "name": "Fred Flintstone",
        "email": "fred@example.com"
      }
    },
    {
      "links": {
        "$self": "https://blog/api/users/2"
      },
      "attributes": {
        "name": "Wilma Flintstone",
        "email": "wilma@example.com"
      }
    }
  ]
}
```

Resource collections can also have `links` and `meta` fields.  We reserve the link names `$first`, `$last`, `$next`, `$previous` to refer to the first, last, next and previous pages respectively in paged responses.

If the caller requests other resources to be embedded (more on that later), or the server automatically embeds resources for performance reasons, then there will be more than one resource in the response.  For example, if the `posts` link was embedded in the above response, it would look like this:

```json
{
  "links": {
    "posts": "https://blog/api/posts?filter[authorId]=1"
  },
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  },
  "meta": {
    "postCount": 10
  },
  "includes": [
    {
      "links": {
        "$self": "https://blog/api/posts?filter[authorId]=1"
      },
      "elements": {
        "https://blog/api/posts/1": {
          "attributes": {
            /* etc */
          }
        }
      }
    }
  ]
}
```

For this reason, a client should check the response before fetching related resources, in case the resource has been fetched already.  Note that the server might choose to just include `meta` or `links` information for a given included resource: thus, if a resource lacks has neither an `attributes` nor an `elements` field, the client should make a request to the URL to get the full resource.  As a slightly contrived example, the `postCount` field above could have been implemented as just a `count` field on the included resource, like so:

```json
{
  "links": {
    "posts": "https://blog/api/posts?filter[authorId]=1"
  },
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  },
  "includes": [
    {
      "links": {
        "$self": "https://blog/api/posts?filter[authorId]=1"
      },
      "meta": {
        "count": 10
      }
    }
  ]
}
```

In this case, in order for the client to actually get the list of posts, it must make the request to the link URL.

## The Error object

If a request causes an error, the response will look like so (e.g., resource not found):

```json
{
  "error": {
    "name": "NotFoundError",
    "message": "The specified resource was not found.",
    "status": 404,
    "meta": {
      "id": 1
    }
  }
}
```

In words, that's an object containing an `error` field.  This field is itself an object, which must have a `name` and a `message` field, containing string values.  It is suggested that for exceptions, these are the exception name and detail respectively.

There may also be a `status` field, which has the HTTP status as an integer.  This might be useful if you have a middleware catching exceptions and using specific HTTP status codes if defined.

Finally, the object could contain a `meta` field for any other relevant information, such as data explaining what specifically went wrong.  For example, in validation errors, this might describe what fields were wrong and what was wrong with them.

A client should be able to assume that if `error` is defined on the response, then something went wrong, so don't use this for any other reason.

## Working with resources

### Fetching

Getting resources is implemented with the `GET` HTTP method, as you would expect.  We suggest the URL `/:resource` is used for collections and `/:resource/:id` is used for single objects.  For convenience, you might choose to also implement related resource URLs: e.g., the URL for all blog posts belonging to user `1` might be `/api/users/1/posts`.  Depending on your implementation, this is probably just going to be an alias for `/api/posts?filter[authorId]=1` (see below for filtering), so you should include the `$self` link on the non-canonical one to point to the canonical URL, as described above.

Various query parameters are available, described below.


#### Inclusion of related resources

A client can specifically elect to include related resources with the `include` query parameter.  Note that for performance reasons the server may decide to include certain related resources anyway without being asked, or it might choose to ignore requests asking for specific resources to be included, so no assumptions should be made by the client about included resources.

A blog post might define `links` called `author` and `comments`, such that `GET /posts/1` would return:

```json
{
  "links": {
    "author": "https://blog/api/authors/1",
    "comments": "https://blog/api/posts/1/comments"
  },
  "attributes": {
    "title": "Hello, world!",
    "content": "This is a test post."
  }
}
```

To include the `author` relation, you would make a request to `GET /api/posts/1?include=author`, which would return:

```json
{
  "links": {
    "author": "https://blog/api/users/1",
    "comments": "https://blog/api/posts/1/comments"
  },
  "attributes": {
    "title": "Hello, world!",
    "content": "This is a test post."
  },
  "includes": [
    {
      "links": {
        "$self": "https://blog/api/users/1"
      },
      "attributes": {
        "name": "Fred Flintstone",
        "email": "fred@example.com"
      }
    }
  ]
}
```

You can include multiple relations by using a comma-separated list, e.g., `GET /api/posts/?include=author,comments`.

Note that if you use this parameter on a resource collection, it will include the named relations for each element, which could lead to *quite a lot* of data being returned.

If a related resource which does not exist is requested, then an HTTP `400` error with sufficient explanation of the problem should be returned.  If the relation does exist but the server doesn't want to support it for whatever reason, it should just be ignored.  This choice has been made so that programmers can catch programming mistakes easily while still allowing servers to be silently stubborn in order to protect bandwidth, processing or whatever else.


#### Sparse fieldsets

A client can specify a subset of fields that it needs to see in the response, using the `fields` query parameter.  This can help with bandwidth by excluding large fields that aren't needed.  For example, when viewing a list of blog posts you possibly don't need the actual post content, which could be pretty lengthy.

Note that the server could silently ignore your restriction and return you all the fields anyway.  It's up to the server implementer whether to throw on unknown field names (in which case an HTTP `400` should be returned), but if you're using a document-oriented database there might not be the concept of "known" and "unknown" field names.

Fieldsets are specified per link name for included resources, or `$self` for the base resource.  In the example above, to get only the titles of blog posts, you would make a request to `GET /api/posts?fields[$self]=title`.  You could additionally include the `author` link and restrict to only the author name by making a request to `GET /api/posts?include=author&fields[$self]=title&fields[author]=name`.


#### Filtering

Resource collections can be filtered; attempts to filter a resource which isn't a collection will result in an HTTP `400`.  The server does not have to implement filtering for all fields or even at all; filtering by unsupported fields should also result in a `400`.

Filtering is effected via the `filter` query parameter. E.g., to get all the posts with `authorId` of `1`, you would make a request to `GET /api/posts?filter[authorId]=1`.  You can filter on more than one field: the criteria are combined with a logical `AND`.  The values of the criteria are to be interpreted as JSON fragments, thus to search for users named 'Bob', you'd make a request to `GET /api/users?filter[name]='Bob'`.  It's up to the application whether to be case sensitive or not for string comparisons.

You can filter using multiple values (analagous to the SQL `IN` keyword) by using a comma-separated list.  E.g., for all posts belonging to authors 1 or 2, a request is made to `GET /api/posts?filter[authorId]=1,2`.

Nested fields would be filtered by putting the child name in square brackets like so: `?filter[parent][child]=value`.


#### Sorting

Similarly, resource collections can be sorted; attempts to sort a resource which isn't a collection will result in an HTTP `400`.  Sorting by unsupported fields should also result in a `400`.

To sort, a comma-separated list of (1 or more) fields to sort by is passed into the `sort` parameter.  To sort descending, the field name is prefixed with a dash (`-`).  E.g., to sort by the `name` field, you would use `?sort=name`, and to sort by `name` and then by `age` descending, you would use `?sort=name,-age`.

#### Paging

Again, resource collections can be paged using the `page` parameter; attempts to page a resource which isn't a collection will result in an HTTP `400`, as should attempts to use a method of paging which the server doesn't support.

There are 3 types of paging:

  * number based: `page[number]` and `page[size]` for regular 1-based index paging

    For example, `?page[number]=1&page[size]=10` will give you rows 0-9, and `?page[number]=4&page[size]=5` will give you rows 14-19.

  * offset based: `page[offset]` and `page[size]`

    The above examples would instead be written as `?page[offset]=0&page[size]=10` and `?page[offset]=14&page[size]=5` respectively.

  * fast paging: `page[after]` and `page[size]` for fast paging like they use on Reddit

    Using this method means you can avoid a costly index/table scan which is useful in huge collections:  `page[after]` is set to a field value and `page[size]` has its usual meaning.  If the collection is by default sorted by ID, then `page[after]` works on the ID field by default.  If there is a custom sort applied, then `page[after]` works on the major sort field (i.e., the first in a list of fields to sort by).

    For example, `?page[after]=5&page[size]=10` would get the 10 rows with ID field greater than `5`, and `?page[after]='Zack'&page[size]=5&sort=-name` would get the 5 rows with name less (by collation, ASCII, whatever is application-defined) than `'Zack'`.


The server is free to pick a sensible default method of paging, and default value for `page[size]`.  It may also choose to ignore `page[size]` altogether, or after a certain maximum page size.  The `$next`, `$previous`, `$first` and `$last` links returned should match the method of paging used by the request.


### Creating

Resources are created by `POST`ing a resource object to a resource collection URL.  E.g., to create a `user` resource, you would post the following body to `POST /api/users`:

```json
{
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  }
}
```

In the response, the server sets the `Location` header to the URL of the newly-created resource.  It can send either an HTTP `204` with an empty body, or it can send an HTTP `201` with the object which was created (ostensibly the same as a `GET` request to the `Location` URL).  The latter can be useful in the case of generated fields.

The server can perform validation on the request body before accepting it: if the validation fails, the server should respond with an HTTP `400` and an error object giving sufficient information as to what caused the validation failure.


### Updating

Resources can be updated by sending a `PATCH` request containing a single resource object to the individual resource URL, or by sending `PATCH` request containing a resource collection object to the resource collection URL.  Sending a collection to a single resource or vice versa will result in an HTTP `409`.

For example, to update user `1`, a request can be made to `PATCH /api/users/1`, with the following body:

```json
{
  "attributes": {
    "name": "Fred Flintstone",
    "email": "fred@example.com"
  }
}
```

The specified fields will then be updated with the specified values.  The same effect could be achieved by sending a collection object to `PATCH /api/users`:

```json
{
  "elements": [
    {
      "links": {
        "$self": "http://blog/api/users/1"
      },
      "attributes": {
        "name": "Fred Flintstone",
        "email": "fred@example.com"
      }
    }
  ]
}
```

If the ID URL is invalid for the request endpoint, an HTTP `409` is returned.

Resources can also be updated using the HTTP `PUT` method: in contrast to `PATCH`, this replaces the entire object to be updated with the request object, rather than updating only the specified fields.


### Deleting

Deleting a resource is simply a matter of making a `DELETE` request to the resource URL.  The server can support deletion of all objects in a collection by supporting `DELETE` requests to the resource collection URL.


### General

The server doesn't have to support all methods above.  If an unsupported method is used, the server responds with an HTTP `405`.

Making a `GET` request to the API route should return a result with links to each of the resources.  E.g. in our blog example, making a request to `GET /api` would return the following document:

```json
{
  "links": {
    "posts": "http://blog/api/posts",
    "users": "http://blog/api/users",
    "comments": "http://blog/api/comments"
  }
}
```
