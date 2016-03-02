
# Super API

Super API will make you super 'appy.

## Overview

Super API is essentially a JSON API specification like [JSON:API](https://jsonapi.org) or [HAL](https://stateless.co/hal_specification.html).  We believe it has advantages over both though (otherwise we wouldn't have bothered).

JSON:API is billed as an "anti-bikeshedding tool", yet leaves quite a lot of detail up to the implementer.  Super API is quite opinionated in an effort to leave less room for interpretation, although it borrows heavily from the former.

### Basic example

We'll use the canonical example of a blog system to whet your appetite.  Below is the sample response to a request for a list of blog posts.

```json
{
  "$self": {
    "links": {
      "$next": "https://blog/api/posts?page[number]=2"
    },
    "elements": {
      "https://blog/api/posts/1": {
        "links": {
          "author": "https://blog/api/users/1",
          "comments": "https://blog/api/posts/1/comments"
        },
        "attributes": {
          "title": "Hello, world!",
          "content": "This is a test post."
        }
      },
      "https://blog/api/posts/2": {
        "links": {
          "author": "https://blog/api/users/1",
          "comments": "https://blog/api/posts/2/comments"
        },
        "attributes": {
          "title": "Test post 2",
          "content": "This is second test post."
        }
      }
    },
    "meta": {
      "count": 8
    }
  },
  "https://blog/api/users/1": {
    "attributes": {
      "name": "Fred Flintstone",
      "email": "fred@example.com"
    }
  },
  "https://blog/api/posts/1/comments": {
    "elements": {
      "https://blog/api/comments/1": {
        "attributes": {
          "content": "This is a comment!"
        }
      }
    }
  },
  "https://blog/api/posts/2/comments": {
    "elements": {}
  }
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

How about collections of resources?  These are given in the `elements` field of the resource object.  Resources are identified by their URL, and the `elements` field is thus a hash of resource objects keyed by URL:

```json
{
  "elements": {
    "https://blog/api/users/1": {
      "attributes": {
        "name": "Fred Flintstone",
        "email": "fred@example.com"
      }
    },
    "https://blog/api/users/2": {
      "attributes": {
        "name": "Wilma Flintstone",
        "email": "wilma@example.com"
      }
    }
  }
}
```

Resource collections can also have `links` and `meta` fields.  We reserve the link names `$first`, `$last`, `$next`, `$previous` to refer to the first, last, next and previous pages respectively in paged responses.

A complete response body consists of at least one resource object, given the ID `$self`.  For example, a request to `GET /users/1` could return:

```json
{
  "$self": {
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
}
```

If the caller requests other resources to be embedded (more on that later), or the server automatically embeds resources for performance reasons, then there will be more than one resource in the response.  For example, if the `posts` link was embedded in the above response, it would look like this:

```json
{
  "$self": {
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
  },
  "https://blog/api/posts?filter[authorId]=1": {
    "elements": {
      "https://blog/api/posts/1": {
        "attributes": {
          /* etc */
        }
      }
    }
  }
}
```

For this reason, a client should check the response before fetching related resources, in case the resource has been fetched already.  Note that the server might choose to just include `meta` or `links` information for a given included resource: thus, if a resource lacks has neither an `attributes` nor an `elements` field, the client should make a request to the URL to get the full resource.  As a slightly contrived example, the `postCount` field above could have been implemented as just a `count` field on the included resource, like so:

```json
{
  "$self": {
    "links": {
      "posts": "https://blog/api/posts?filter[authorId]=1"
    },
    "attributes": {
      "name": "Fred Flintstone",
      "email": "fred@example.com"
    }
  },
  "https://blog/api/posts?filter[authorId]=1": {
    "meta": {
      "count": 10
    }
  }
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
  "$self": {
    "links": {
      "author": "https://blog/api/authors/1",
      "comments": "https://blog/api/posts/1/comments"
    },
    "attributes": {
      "title": "Hello, world!",
      "content": "This is a test post."
    }
  }
}
```

To include the `author` relation, you would make a request to `GET /api/posts/1?include=author`, which would return:

```json
{
  "$self": {
    "links": {
      "author": "https://blog/api/users/1",
      "comments": "https://blog/api/posts/1/comments"
    },
    "attributes": {
      "title": "Hello, world!",
      "content": "This is a test post."
    }
  },
  "https://blog/api/users/1": {
    "attributes": {
      "name": "Fred Flintstone",
      "email": "fred@example.com"
    }
  }
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


#### Paging
