---
layout: post
title: Accessing either form or query parameters from a single annotated method parameter in Jersey
---

In [Java Jersey](http://jersey.java.net/) (the [JSR 311: JAX-RS](http://jcp.org/en/jsr/detail?id=311) reference implementation), you add Java Annotations to your POJOs to turn them in to RESTful resources. For example:

```java
@Path("/posts")
public class PostsResource {
  @POST
  @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
  public void post(@FormParam("title") String title, @FormParam("body") String body) {
    // Save new post 
  }
}
```

I came across [this post](http://stackoverflow.com/questions/8720728/in-jersey-can-i-combine-queryparams-and-formparams-into-one-value-for-a-method) on StackOverflow, asking for a way to combine form params and query params in a single resource `@POST` method. In other words, the poster wanted to know if he could do this:

```java
@POST 
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
public void post(@BothParam("title") String title, @BothParam("body") String body) {
  // Save new post
}
```

The annotation `@BothParam` would allow you to create a post with either:
  
```bash
curl -d title=mytitle&body=mybody http://localhost:9998/posts
```

or:

```bash
curl -d body=mybody http://localhost:9999/posts?title=title
```

I'm also interested in a good solution to this question. I'm working on a project where I'd like to use Java Jersey, but need to support a legacy API which does not map elegantly into the RESTful paradigms that JAX-RS is designed to support.

One solution I'm exploring is an `InjectableProvider` which provides an `Injectable` for a custom annotation, such as `BothParam`.

The custom annotation, `BothParam`:

```java
@Target({ ElementType.PARAMETER, ElementType.METHOD, ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface BothParam { 
  String value(); 
}
```

The injectable and injectable provider, `BothParamInjectableProvider`:

```java
@Provider
public class BothParamInjectableProvider implements Injectable<String>, implements Injectable<BothParam, Type> {
  @Context HttpContext httpContext;

  private String parameterName;

  public BothParamInjectableProvider(@Context HttpContext httpContext) {
    this.httpContext = httpContext;
  }

  public String getValue() {
    if (httpContext.getRequest().getQueryParameters().containsKey(parameterName)) {
      return httpContext.getRequest().getQueryParameters().getFirst(parameterName);
    } else if (httpContext.getRequest().getFormParameters().containsKey(parameterName)) {
      return httpContext.getRequest().getFormParameters().getFirst(parameterName);
    }
    return null;
  }

  public ComponentScope getScope() {
    return ComponentScope.PerRequest;
  }

  public Injectable getInjectable(ComponentContext componentContext, BothParam bothParam, Type type) {
    this.parameterName = bothParam.getValue;
    return this;
  }
}
```
