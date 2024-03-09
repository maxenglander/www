---
layout: post
title: Constraining Strings in Jersey requests
---

# Constraining Strings in Jersey requests

In this post I'll look at ways to constrain String parameters in [Jersey](http://jersey.java.net/) requests.

## Motivation

Take the following [JSR 311](http://jcp.org/en/jsr/detail?id=311) resource for modifying accounts:

```java
@Path("/accounts")
public class AccountsResource {
  @PUT
  @Path("/{id}")
  @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
  public void modifyAccount(@PathParam("id") Long id, @FormParam("fullName") String fullName,
          @FormParam("status") String status) {
    /* Modify the account */
  }
}
```

How can I constrain `status` to a small set of allowable values, such as "active", "banned", and "disabled"?

## Design goals

Given a String request parameter (such as `status`), and a set of allowable values (such as "active", "banned" and "disabled"),

 * Adding a new allowable value (such as "unconfirmed") does not require updates to the code responsible for constraining `status`
 * Given another request parameter (such as `notification_preference`), and a set of allowable values (such as "daily", "weekly", "monthly", "never"),
   the code responsible for constraining `status` can be re-used to constrain `notification_preference`

## What I wish I could do

I wish I could define an Enum called `AccountStatus`, and have Jersey inject the `status` field of the request into that Enum.

```java
public enum AccountStatus {
  Active("active"),
  Banned("banned"),
  Disabled("disabled");

  private String status;

  private AccountStatus(String status) {
    this.status = status;
  }
  
  public String toString() {
    return this.status;
  }
}

@Path("/accounts")
public class AccountsResource {
  @PUT
  @Path("/{id}")
  @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
  public void modifyAccount(@PathParam("id") Long id, @FormParam("fullName") String fullName, @FormParam("status") AccountStatus status) {
    /* Modify the account */
  }
}
```

Unfortunately, this isn't supported by Jersey, or any other JSR 311 implementor, as far as I know.

## Method 1: A straight-forward way

A straight-forward way is to simply check the value of `status` against each allowable value, 
and raise a `WebApplicationException` if it is not within any of the allowable set of values:

```java
@Path("/accounts")
public class AccountsResource {
  public static final String STATUS_ACTIVE = "active";
  public static final String STATUS_BANNED = "banned";
  public static final String STATUS_DISABLED = "disabled";

  @PUT
  @Path("/{id}")
  @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
  public void modifyAccount(@PathParam("id") Long id, @FormParam("fullName") String fullName, @FormParam("status") String status) throws WebApplicationException {
    validateStatus(status);
    /* Modify the account */
  }

  public static void validateStatus(String status) {
    if(!STATUS_ACTIVE.equals(status)
      && !STATUS_BANNED.equals(status)
      && !STATUS_DISABLED.equals(status)) {
      throw new WebApplicationException(
        Response.status(Status.BAD_REQUEST)
                .entity("status must be one of [" + STATUS_ACTIVE + ", " + STATUS_BANNED + ", " + STATUS_DISABLED + "]")
                .build();
      );
    }
  }
}
```

It should be apparent that this approach fails the design goals. This might be perfectly acceptable for a simple application. However, this approach
could become cumbersome in larger applications with lots of resources constraining Strings to small sets of allowable values.

## Method 2: Use Enums

We can define an Enum for any String parameter we want to constrain:

```java
public enum AccountStatus {
  Active("active"),
  Banned("banned"),
  Disabled("disabled");

  private String status;

  private AccountStatus(String status) {
    this.status = status;
  }

  public String toString() {
    return this.status;
  }
}
```

And constrain any String to any Enum with a utility method like the following `containsString` method:

```java
public class EnumUtil {
  public static boolean containsString(Class<? extends Enum> enumClass, String string) {
    String[] stringValues = stringValues(enumClass);
    for(int i = 0; i < stringValues.length; i++) {
      if(stringValues[i].equals(string)) {
        return true;
      }
    }
    return false;
  }

  public static String[] stringValues(Class<? extends Enum> enumClass) {
    EnumSet enumSet = EnumSet.allOf(enumClass);
    String[] values = new String[enumSet.size()];
    int i = 0;
    for(Object enumValue : enumSet) {
      values[i] = String.valueOf(enumValue);
      i++;
    }
    return values;
  }
}
```

Making our resource look like this:

```java
@Path("/accounts")
public class AccountsResource {
  @PUT
  @Path("/{id}")
  @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
  public void modifyAccount(@PathParam("id") Long id, @FormParam("fullName") String fullName, @FormParam("status") String status) throws WebApplicationException {
    validateWithEnum("status", AccountStatus.class, status);
    /* Modify the account */
  }

  public static void validateWithEnum(String parameterName, Class<? extends Enum> enumClass, String parameterValue) {
    if(!EnumUtil.containsValue(enumClass, parameterValue)) {
      throw new WebApplicationException(
        Response.status(Status.BAD_REQUEST)
                .entity(parameterName + " must be one of " + EnumUtil.stringValues(enumClass))
                .build();
      );
    }
  }
}
```

This approach satisfies the design goals. Adding a new member to `AccountStatus` (like "unconfirmed") requires a one-line change.
The `validateWithEnum` method could be pushed down to a base class, or extracted to a separate class, and then re-used by any
JSR 311 resource method that constrains parameter values to Enums.
