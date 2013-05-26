---
layout: post
title: Validating Jersey 1.x requests
---

[Jersey](http://jersey.java.net/) 1.x does not provide validation of request parameters - it leaves that up to you.
Jersey 2.x [*does*](https://github.com/jersey/jersey/commit/7b8301f8c2eb81fe8e304ac5108a547184e67ee0) provide validation 
of request parameters. However, until 2.x becomes stable, I feel more comfortable using 1.x and doing validation on my own.

Starting from an unvalidated resource, I'll sketch out several increasingly generalized approaches to validating request parameters.

# An unvalidated resource

Consider the following JSR 311 resource:

{% highlight java %}
  @Path("/posts")
  public class PostsResource {
    public static class SearchParams {
      // Search for posts that were posted in the given year
      // E.g. to search for posts posted in 2003
      //   GET /posts?year=2003
      @QueryParam("year")
      public Integer year;

      // E.g. to search for posts posted in March
      //   GET /posts?month=3
      @QueryParam("month")
      public Integer month;

      // Valid states: draft, published, deleted
      // E.g. to search for posts that are either drafts or deleted
      //   GET /posts?state=draft&state=deleted
      @QueryParam("state")
      public String[] states;
    }

    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    public List<Post> search(@InjectParam SearchParams searchParams) {
      // Search for and return posts that meet SearchParam's criteria
      return runDatabaseQuery(searchParams);
    }

    protected List<Post> runDatabaseQuery(SearchParams searchParams) { 
      List<Post> posts = new ArrayList<Post>();
      // TODO: Run database query, and populate list of posts
      return posts;
    }
  }
{% endhighlight %}

# Approaches to validation

All of the approaches meet the same essential requirements:

* Do not query the database if any request parameter is invalid
* Return errors messages to the client if any request parameter is invalid

What varies between approaches is which components take responsibility for the following concerns:

* Expression of constraints
* Performing validation
* Generating and returning error responses

## Do everything in the `SearchParams` class

{% highlight java %}
  @Path("/posts")
  public class PostsResource {
    public static class SearchParams {
      public static List<String> VALID_STATES = Arrays.asList(new String[] { "draft", "published", "deleted" });
      public static int MIN_YEAR = 1970;
      public static int MIN_MONTH = 1;
      public static int MAX_MONTH = 12;

      @QueryParam("date")
      public Map<String, String> date;

      @QueryParam("state")
      public String[] states;

      public List<String> validate() {
        List<String> errorMessages = new ArrayList<String>();

        // Validate states
        for(String state : states) {
          if(!VALID_STATES.contains(state)) {
            errorMessages.add(state + " is not a valid state");
          }
        }

        // Validate year and month
        if (year < MIN_YEAR) {
          errorMessages.add("there are no posts older than " + SearchParams.MIN_YEAR);
        }

        // Validate month
        if (month < MIN_MONTH || month > MAX_MONTH) {
          errorMessages.add(month + " is not a valid month);
        }

        return errorMessages;
      }
    }

    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    public List<Post> search(@InjectParam SearchParams searchParams) {
      List<String> errorMessages = searchParams.validate();
      if(!errorMessages.isEmpty()) {
        throw new WebApplicationException(
          Response.status(Status.BAD_REQUEST)
                  .entity(errorMessages)
                  .build();
        );
      }
      return runDatabaseQuery(searchParams);
    }
  ...
{% endhighlight %}

I think this approach is good for simple applications. For applications with lots of resources, with complex
validation requirements, or with resources with shared validation requirements, it may make sense to generalize
validation.

To see how this approach could become cumbersome, imagine having several other REST methods in `PostsResource`,
each with their own parameter object, with several `Integer` fields that we want to apply a minimum and maximum
constraint to. In that scenario, each parameter object would have similar validation logic as the `validate`
method of `SearchParams`. It would be much better if we could generalize the application of and validation of
minimum and maximum constraints.

## Use JSR 303 bean validation

[JSR 303](http://beanvalidation.org/1.0/spec/) bean validation is awesome. It allows you to apply constraints to POJOs
with annotations, and validate them in a uniform way with a validator class. Here I declare constraints directly on
`SearchParams`.

{% highlight java %}
  @Path("/posts")
  public class PostsResource {
    public static class SearchParams {
      @QueryParam("year")
      @Min(1970)
      public Integer year;

      @QueryParam("month")
      @Min(1)
      @Max(12)
      public Integer month;

      @QueryParam("state")
      @Pattern.List({@Pattern(regexp="^draft$"), @Pattern("^published$"), @Pattern("^deleted$")}
      public String[] states;

      public List<String> validate() {
        List<String> errorMessages = new ArrayList<String>();
        Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

        Set<ConstraintViolation<SearchParams>> violations = validator.validate(this);
        if (!violations.empty) {
          for(ConstraintViolation<SearchParams> violation : violations) {
            errorMessages.add(violation.getMessage());
          }
        }

        return errorMessages;
      }
    }

    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    public List<Post> search(@InjectParam SearchParams searchParams) {
      List<String> errorMessages = searchParams.validate();
      if(!errorMessages.isEmpty()) {
        throw new WebApplicationException(
          Response.status(Status.BAD_REQUEST)
                  .entity(errorMessages)
                  .build();
        );
      }
      return runDatabaseQuery(searchParams);
    }
  ...

{% endhighlight %}

The `validate` method is no longer doing anything specific to `SearchParams`. Therefore, it could be factored to a base
class. The validator factory should also be re-used, rather than recreated each time `validate` is called.

## (Almost) removing validation from resources

In all of the previous approaches, the `PostsResource` takes some responsibility for validation. It initiates the
validation process, and generates an error response if validation fails. These two responsibilities are not specific
to either `PostsResource` or to `SearchParams`, and can be factored out.

### The Dropwizard way

[dropwizard](http://dropwizard.codahale.com/) bundles (among other things) 
[Hibernate Validator](http://www.hibernate.org/subprojects/validator.html) (an implementation of JSR 303) and Jersey.

Using Dropwizard, we could remove the `validate` method from `SearchParams` entirely, and simply add a `@Valid`
annotation in front of the `SearchParams` parameter in the `search` method of `PostsResource`:

{% highlight java %}
  @Path("/posts")
  public class PostsResource {
    public static class SearchParams {
      @QueryParam("year")
      @Min(1970)
      public Integer year;

      @QueryParam("month")
      @Min(1)
      @Max(12)
      public Integer month;

      @QueryParam("state")
      @Pattern.List({@Pattern(regexp="^draft$"), @Pattern("^published$"), @Pattern("^deleted$")})
      public String[] states;
    }

    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    public List<Post> search(@InjectParam @Valid SearchParams searchParams) {
      return runDatabaseQuery(searchParams);
    }
  ...

{% endhighlight %}

Dropwizard uses a [custom subclass](https://github.com/codahale/dropwizard/blob/c0cec2cef90ea805c409e94aca6dcec204cf5e14/dropwizard-core/src/main/java/com/yammer/dropwizard/jersey/JacksonMessageBodyProvider.java) 
of `JacksonJaxbJsonProvider` to check for the presence of `@Valid` annotations on resource method params, 
validate them, and generate an error response if validation fails.

This comes with its limitations. For example, Dropwizard doesn't give you a way to customize the generation of error responses.

### The AOP way

Using aspect-oriented programming, we can achieve an outcome similar to Dropwizard's: validation occurs
before resource methods are executed.

{% highlight java %}
  @Aspect
  public class ValidationAspect {
    private Validator validator;

    public ValidationAspect() {
      this.validator = Validation.buildDefaultValidatorFactory().getValidator();
    }

    @Before("@annotation(javax.validation.Valid)")
    public void validate(JoinPoint joinPoint) {
      List<String> errorMessages = new ArrayList<String>();
      for(Object parameter : joinPoint.getArgs()) { 
        Set<ConstraintViolation<Object>> violations = validator.validate(parameter);
        for(ConstraintViolation<Object> violation : violations) {                                           
          errorMessages.add(violation.getMessage());
        }
      }

      if(!errorMessages.empty()) {
        throw new WebApplicationException(
          Response.status(Status.BAD_REQUEST)
                  .entity(errorMessages)
                  .build();
        );
      }
    }
  }

  @Path("/posts")
  public class PostsResource {
    public static class SearchParams {
      @QueryParam("year")
      @Min(1970)
      public Integer year;

      @QueryParam("month")
      @Min(1)
      @Max(12)
      public Integer month;

      @QueryParam("state")
      @Pattern.List({@Pattern(regexp="^draft$"), @Pattern("^published$"), @Pattern("^deleted$")})
      public String[] states;
    }

    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    @Valid 
    public List<Post> search(@InjectParam SearchParams searchParams) {
      return runDatabaseQuery(searchParams);
    }
  ...


{% endhighlight %}

I don't understand AOP very well, and this is probably overkill for most applications.

However, this approach gives you a ton of flexibility. I'll flesh it out in another post.
