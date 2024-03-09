---
layout: post
title: Basic RESTful API versioning in Jersey
---

# Basic RESTful API versioning in Jersey

Have you ever wondered how you'd create a versioned API in [Jersey](http://jersey.java.net) using the `Accept` HTTP header? If so, read on.

I won't go into details about how and why you would use the `Accept` header to accomplish versioning. That's covered very well [elsewhere](http://barelyenough.org/blog/2008/05/versioning-rest-web-services/).

Pretend we're an online music store, and are building an API.

## Version 1

In the first incarnation of the app, we create a `TrackResource` to serve our collection of tracks.

```java
// Because we're seasoned API developers, we version
// our resource representions from the very start
public class TrackV1 {
    private final String artistName;
    private final String title;
    private final String length;
    private final int year;

    public Track(String artistName, String title, String length, int year) {
        this.artistName = artistName;
        this.title = title;
        this.length = length;
        this.year = year;
    }

    public String getArtistName() {
        return artistName;
    }

    public String getLength() {
        return length;
    }

    public String getTitle() {
        return title;
    }

    public int getYear() {
        return year;
    }
}
```

Using the `@Produces` annotation, we can determine which representation gets returned when a client requests a particular version of our API.

```java
@Resource
@Path("/track")
@Produces("application/vnd.musicstore-v1+json")
public class TrackResource {
    @GET
    @Path("/{id}")
    public TrackV1 getV1(@PathParam("id") int id) {
        return new TrackV1("Woody Guthrie", "Jackhammer John", "2:30", 1941);
    }
}
```

Using `curl`, a client would request the version 1 representation of a track as follows:

    curl -H "Accept: application/vnd.musicstore-v1+json" http://localhost:8080/track/1

and would receive the following JSON response:

    {"artistName":"Woodie Guthrie","length":"2:30","title":"Jackhammer John","year":1941}

## Version 2

As our product evolves, we come up with a lot of ideas to improve the API. Any changes we want to make that would
break the API for current clients, we put in version 2 of the API. Here are some (arbitrary) examples of non-backwards-compatible
changes:

 * Rename `artistName` to `artist`
 * Deprecate `year`
 * Change the type of `length` from a `String` to an `int`

```java
public class TrackV2 {
    private final String artist;
    private final String title;
    private final int length;

    public TrackV2(String artist, String title, int length) {
        this.artist = artist;
        this.title = title;
        this.length = length;
    }

    public String getArtist() {
        return artist;
    }

    // Length of the track in seconds
    public int getLength() {
        return length;
    }

    public String getTitle() {
        return title;
    }
}
```

In `TrackResource`, we add a new method to return this representation of a track when requested.

```java
    ...

    @GET
    @Path("/{id}")
    @Produces("application/vnd.musicstore-v2+json")
    public TrackV2 getV2(@PathParam("id") int id) {
        return new TrackV2("Woodie Guthrie", "Jackhammer John", 150)
    }

    ...
```

Using `curl`, a client would request the version 2 representation of a track as follows:

    curl -H "Accept: application/vnd.musicstore-v2+json" http://localhost:8080/track/1

and would receive the following JSON response:

    {"artistName":"Woodie Guthrie","length":150,"title":"Jackhammer John"}

## Code re-use

As it is, the amount of code duplicated between `TrackV2` and `TrackV1` isn't *that* bad. However, it could be worse. If it were much worse,
we could use a combo of subclassing and [Jackson](http://wiki.fasterxml.com/JacksonHome) (JSON generator used by Jersey) annotations to de-duplicate.

`TrackV1` becomes:

```java
public class TrackV1 {
    private final String artistName;
    private final String title;
    private final String length;
    private final int year;

    public Track(String artistName, String title, String length, int year) {
        this.artistName = artistName;
        this.title = title;
        this.length = length;
        this.year = year;
    }

    public String getArtistName() {
        return artistName;
    }

    public Object getLength() {
        return length;
    }

    public String getTitle() {
        return title;
    }

    public int getYear() {
        return year;
    }
}
```

while `TrackV2` becomes:

```java
public class TrackV2 extends TrackV1 {
    private final int length;

    public TrackV2(String artistName, String title, int length, int year) {
        super(artistName, title, (length / 60) + ":" + (length % 60), year);
        this.length = length;
    }

    @Override
    @JsonProperty("artist")
    public String getArtistName() {
        return super.getArtistName();
    }

    @Override
    public Object getLength() {
        return length;
    }

    @Override
    @JsonIgnore
    public int getYear() {
        return super.getYear();
    }
}
```

The key features of this de-duplication are:

  * `@JsonProperty`, a Jackson annotation which allows you to control the name of the key to which a POJO field or method
  is serialized
  * `@JsonIgnore`, a Jackson annotation which should be pretty self-explanatory
  * The change of `getLength`'s return type from a `String` to an `Object`
    * Jackson is smart enough to detect the type of the return value, and serializes accordingly

## Almost perfect

One thing that annoys me about [JAX-RS](https://jax-rs-spec.java.net/) (the spec for which Jersey is the reference implementation)
is that you can't have two different resources for the same path. I wish I could organize my API versions into multiple resource
classes, each responsible for handling a different version level. 

Granted, I could just use path-based versioning (e.g. `/v1/track` and `/v2/track`) to circumvent this restriction, but I'm sold on 
some of the [arguments](http://stackoverflow.com/questions/389169/best-practices-for-api-versioning) for header-based versioning.

Any how, I wish I could do this:

### Version 1

```java
@Resource
@Path("/track")
public class TrackResourceV1 {
    @GET
    @Path("/{id}")
    @Produces("application/vnd.musicstore-v1+json")
    public TrackV1 get(@PathParam("id") int id) {
        /* As above */
    }
}
```

### Version 2

```java
@Resource
@Path("/track")
public class TrackResourceV2 {
    @GET
    @Path("/{id}")
    @Produces("application/vnd.musicstore-v2+json")
    public TrackV2 get(@PathParam("id") int id) {
        /* As above */
    }
}
```

But I can't.

## Code

[Jersey versioning example](https://github.com/maxenglander/jersey-versioning-example/tree/v1.0)
