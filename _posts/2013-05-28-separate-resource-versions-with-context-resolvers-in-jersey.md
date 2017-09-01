---
title: Separate resource classes for RESTful API versioning in Jersey
layout: post
---

# Separate resource classes for RESTful API versioning in Jersey

In my [last post](/2013/04/23/basic-restful-api-versioning-in-jersey.html), I
looked at ways to implement versioned APIs in
[JSR-311](http://jcp.org/en/jsr/detail?id=311), and lamented the fact that the
spec does not allow you to have two root resource classes with the same `@Path`
but different producible media-types. In other words, 

## I wish you could do this

### Version 1

```java
@Resource
@Path("/track")
@Produces("application/vnd.musicstore-v1+json")
public class TrackResourceV1 {
    @GET
    @Path("/{id}")
    public TrackV1 get(@PathParam("id") int id) {
        // Do v1 stuff
    }
}
```

### Version 2

```java
@Resource
@Path("/track")
@Produces("application/vnd.musicstore-v2+json")
public class TrackResourceV2 {
    @GET
    @Path("/{id}")
    public TrackV2 get(@PathParam("id") int id) {
        // Do v2 stuff
    }
}
```

But I can't. [Jersey](https://jersey.java.net/), the JSR-311 reference
implementation, will complain of multiple root resources with the same path, and
fail to start up.

## Have root resource delegate to resource versions

While we can't have two root resource classes with the same path, we can have
one root resource delegate work to other (non-root) resource classes based on
the requested media-type.

### Rough picture

```java
@Path("/track")
public class TrackResourceDelegator implements TrackResource {
    public Track get(int id) {
        TrackResource delegate = getDelegateBasedOnRequestedMediaType();
        delegate.get(id);
    }
}

@Produces({"application/vnd.musicstore-v1+json", "application/vnd.musicstore-v2+json"})
public interface TrackResource {
    @GET
    @Path("{id}")
    Track get(@PathParam("id") int id);
}
```

The above is meant to only give a rough picture, and does not explain how we
actually acquire a `TrackResource` delegate based on the requested media type.
There are probably a few different ways this could be accomplished.

### Getting the right delegate

One way to get the right delegate is to use the `Providers` interface to request
a `TrackResource` based on the acceptable media types specified by the request.

```java
@Resource
@Path("/track")
public class TrackResourceDelegator implements TrackResource {
    private TrackResource delegate;

    public TrackResourceDelegator(@Context Providers providers, @Context HttpHeaders httpHeaders) {
        // Iterate over all acceptable requested media types, from the most acceptable to
        // the least acceptable
        for(MediaType mediaType : httpHeaders.getAcceptableMediaTypes()) {
            // Get a TrackResource ContextResolver for the media type
            ContextResolver<TrackResource> trackResourceContextResolver = providers.getContextResolver(TrackResource.class, mediaType); 
            // If there is a ContextResolver capable of providing such a TrackResource, use it
            if (trackResourceContextResolver != null) {
                trackResource = trackResourceContextResolver.getContext();
                break;
            }
        }
    }

    public Track get(int id) {
        delegate.get(id);
    }
}
```

The above constructor will get called for every request. Each request will come
with an acceptable media type. Because of the `@Produces` annotation on the
`TrackResource` interface, only requests containing one of the producible media
types will be allowed through - all others will be rejected with an `HTTP 406`.

All that remains now is to set up our `TrackResource` delegates, and make sure
they are properly provided via `ContextResolvers`.

### Providing delegates with context resolvers

We can simply make our `TrackResource` implementations `@Providers` of
themselves. In Jersey, we have to register these classes as providers, using
scanning resource config or manually adding them to the resource config. 

```java
@Resource
@Produces("application/vnd.musicstore-v1+json")
public class TrackV1Resource implements TrackResource, ContextResolver<TrackResource> {
    public Track get(int id) {
        // Do v1 stuff
    }

    public TrackResource getContext(Class<?> type) {
        return this;
    }
}
```

We do the same thing for version two.

## Comments

There are probably better ways to provide `TrackResource` and acquire
implementations than with `ContextResolvers` and the `Providers` interface.  For
example, Jersey provides `InjectableProviders` which might be more elegant than
my solution. I chose the above route because I wanted to come up with the most
portable solution (using only the tools provided by JSR-311).

I rather like having a resource definition in an interface, and its
implementation spread across a delegator and multiple delegates. However, I
recognize that it's a little burdensome to manage, and not immediately clear
where `@Path` and `@Produces` annotations belong (or don't belong). For example,
I instinctively want to put `@Path` on the `TrackResource` interface, but this
will not do. Each `TrackResource` implementation will pick up the annotation,
and Jersey will complain about multiple resources with the same `@Path`.

## Code

[https://github.com/maxenglander/jersey-versioning-example/tree/v1.1](https://github.com/maxenglander/jersey-versioning-example/tree/v1.1)
