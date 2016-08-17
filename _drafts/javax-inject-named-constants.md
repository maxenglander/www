---
title: Keeping Safe and DRY with Guice and `@Named` Injections
layout: post
---

# Keeping Safe and DRY with Guice and Javax `@Named` Injections

[JSR-330](https://jcp.org/en/jsr/detail?id=330) specifies a `@Named` annotation
that can be used to inject values by `String` keys. Combined with the `bindProperties`
method of Guice [`Names`](http://google.github.io/guice/api-docs/latest/javadoc/index.html?com/google/inject/Binder.html),
we can easily inject configuration values by key name.

A little extra work is needed to make sure that configuration values are validated,
and that the key names are defined in only one place.

## A quick example

Here is a quick example of injecting configuration values into Guice-managed code. I use
`System.getProperties()` here, but `String[] args` could be used just as well.

{% highlight bash %}
$ java -Dmy.value=abc123 -jar my.jar
{% endhighlight %}

{% highlight java %}
package com.maxenglander.examples;

import com.google.inject.Guice;
import com.google.inject.Injector;

public class Cli {
    public static void main(String []) {
        MyModule myModule = new MyModule(System.getProperties());
        Injector injector = Guice.createInjector(myModule);         
        MyApp myApp = injector.getInstance(MyApp.class);  

        System.out.println(myApp.getValue());
    }
}
{% endhighlight %}

{% highlight java %}
package com.maxenglander.examples;

import com.google.inject.AbstractModule;
import com.google.inject.name.Names;

public class MyModule extends AbstractModule {
    public MyModule(Properties properties) {
        this.properties = properties;
    }

    @Override
    public void configure() {
        Names.bindProperties(binder(), this.properties);
    }
}
{% endhighlight %}

{% highlight java %}
package com.maxenglander.examples;

import javax.inject.Inject;
import javax.inject.Named;

class MyApp {
    private final String value;

    @Inject
    MyApp(@Named("my.value") String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
{% endhighlight %}

## What is the danger? 

What is risky about all of this? The danger is that, if `-Dmy.value` is omitted for any reason,
Guice will not like it at all. Some not-so-useful exception will be thrown, and the program will
exit.

Validation (safety) can be as simple as checking for the presence of `my.value` in the `main` method:

{% highlight java %}
if (System.getProperties("my.value") == null) {
    System.out.println("my.value property is required");
    return;
}
{% endhighlight %}

## What is that smell?

At this point, `"my.value"` is present in two places. The quick fix to DRY it up is to store it
in a constant field somewhere (in `MyApp`, or a separate `Configuration` class, for instance).

{% highlight java %}
package com.maxenglander.examples;

class Configuration {
    private Configuration() {}

    public static final String MY_VALUE = "my.value";
}
{% endhighlight %}

{% highlight java %}
if (System.getProperties(Configuration.MY_VALUE) == null) {
    System.out.println("my.value property is required");
    return;
}
{% endhighlight %}

{% highlight java %}
@Inject
MyApp(@Named(Configuration.MY_VALUE) String value) {
    this.value = value;
}
{% endhighlight %}

This approach gets annoying after a few cycles of this:

1. A new configuration value `my.anotherValue` is introduced
1. A new constant field named `ANOTHER_VALUE` is added to `Configuration`
1. A new `if` block is introduced to validate the presence of
   `ANOTHER_VALUE` in the program arguments (`System.getProperties()`)

## Iterating over configuration keys

To make things DRY again, what is needed is a way to iterate over all
configuration keys, and apply the same validation logic to each.

The next logical (at least to me) step is to store configuration keys
in an enumerable. There are a few ways this could be done.

### Using an `enum` (will not work)

Using an `enum` would be lovely (but will not work).
`enum`s are type-safe, enumerable, and can easily be used to
mark a configuration key with metadata, such as whether it
should be required, have a default value, etc.

{% highlight java %}
class Configuration {
    enum Key {
        ANOTHER_VALUE("my.anotherValue", true),
        VALUE("my.value", false);

        private final String name;
        private final boolean required;

        public Key(String name, boolean required) {
            this.name = name;
            this.required = required;
        }

        public String getName() {
            return this.name;
        }

        public boolean isRequired() {
            return this.required;
        }
    }

    private Configuration() {
    }
}
{% endhighlight %}

Sadly, we cannot use an `enum` directly in `@Named`. The following will generate
a compiler error about needing a constant value for `@Named`.

{% highlight java %}
@Inject
MyApp(@Named(Configuration.Key.VALUE.getName()) String value) {
    this.value = value;
}
{% endhighlight %}

I have searched near and far for ways to use `enum`s directly in `@Named`.
I have tried casting the `enum` as a `String`, creating an implementation
of `@Named` that accepts an `enum value()`, having `enum Key implement Named`,
all in vain.

### With a field and an enum

We can combine the original approach of storing key names in static fields
with an `enum` for iterating over keys and, perhaps, storing key metadata.

{% highlight java %}
class Configuration {
    public final String ANOTHER_VALUE = "my.anotherValue";
    public final String VALUE = "my.value";

    enum Key {
        ANOTHER_VALUE(Configuration.ANOTHER_VALUE, true),
        VALUE(Configuration.MY_VALUE, false);

        private final String name;
        private final boolean required;

        public Key(String name, boolean required) {
            this.name = name;
            this.required = required;
        }

        public String getName() {
            return this.name;
        }

        public boolean isRequired() {
            return this.required;
        }
    }

    private Configuration() {}
}
{% endhighlight %}

{% highlight java %}
@Inject
MyApp(@Named(Configuration.MY_VALUE) String value) {
    this.value = value;
}
{% endhighlight %}

## Not lazy enough

With the constant-field-and-enum approach, two smells have been dried up:

 * Duplicate inline key names (e.g. "my.value") with constant `String` fields in `Configuration`
 * Storing constant key names in an enumerable type will allow us to de-duplicate validation logic

Something still does not feel right, however.

Defining a `MY_VALUE` field an a `MY_VALUE` `enum` member is a double-definition.
An absent-minded developer like myself is liable to forget to create a new `enum` member when 
defining `YET_ANOTHER_VALUE`. Granted, unit tests can provide a safety-net for this situation.
However, relying on tests to point out my absent-mindedness means I spend extra time
(fixing the test I broke), and feel more stupid than usual.

What is needed is something that does the heavy-lifting of adding configuration key names
(stored in constant fields) to an enumerable.
