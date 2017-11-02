---
title: Using H2 instead of MySQL for functional tests
layout: post
---

# Using H2 instead of MySQL for functional tests

A Java application I develop has a data access layer that interacts
with MySQL, via Hibernate. Most of the code in this application I am
content to test with unit tests, but, for the DAL layer, I find it
much more valuable to have tests that exercise an actual database.

There's more than one way to connect a test suite to a database:
start up a MySQL instance before the tests run, or have the
test suite start up an instance, perhaps using something like
[this](https://github.com/wix/wix-embedded-mysql). Another option,
the one I've chosen, is to use a pure Java, in-memory database with
(limited) MySQL compatibility: [H2](https://h2database.com).

I can't say that I have a particularly good justification for choosing
H2 over a real MySQL instance. It appeared at the time I chose it to be the
simplest option by virtue of being written in Java and fully in-memory,
meaning it doesn't require interaction with processes or filesystems
outside the JVM, and should be fast (in theory). In retrospect, I think
I should have given more careful consideration to using something like
[wix-embedded-mysql](https://github.com/wix/wix-embedded-mysql) in
a [tmpfs](https://en.wikipedia.org/wiki/Tmpfs).

However, let's pretend both I have good reasons for using H2, and
talk more about using it as a substitute for MySQL in functional tests.

## Connecting to H2 in a test

Assuming that your application uses Hibernate to connect to to MySQL,
then connecting to H2 in a test is, at a minimum, a matter of ensuring
the H2 is in the classpath, and configuring a few properties:

```properties
hibernate.dialect=org.hibernate.dialect.H2Dialect
javax.persistence.jdbc.driver=org.h2.Driver
javax.persistence.jdbc.url=jdbc:h2:mem:test?MODE=MYSQL
```

The URL `jdbc:h2:mem:test?MODE=MYSQL` will instruct Hibernate to use
the H2 JDBC driver to create an in-memory database named "test", and
configure H2 to handle some MySQL-specific syntax. H2 supports other
memory-based database types, such as `memFS`, which will be useful
for our purposes.

How you supply the above properties to Hibernate will depend on how
you've integrated Hibernate into your application. In my case, I use
Google Guice with the guice-persist extension. In my tests I configure
`JpaPersistModule` with these same properties:

```java
Properties jpaProperties = new Properties();
jpaProperties.put("hibernate.dialect", "org.hibernate.dialect.H2Dialect");
jpaProperties.put("javax.persistence.jdbc.driver", "org.h2.Driver");
jpaProperties.put("javax.persistence.jdbc.url", "jdbc:h2:mem:test?MODE=MYSQL");
JpaPersistModule jpaModule
    = new JpaPersistModule('the-name-of-a-unit-defined-in-persistence.xml');
jpaModule.properties(jpaProperties);
Injector injector = Guice.createInjector(jpaModule/*, other modules */);
```

## Seeding and cleaning

Your use case may vary, but there's a chance that, like me, you'll
want to fill H2 with your application's database schema once, add
test-specific fixtures for certain test groups, and ensure that whatever
changes a given test makes to the database do not carry over to the
next test.

In my test suite, I achieve this by doing the following:

 1. Before any tests run, run all of my schema migrations against
    a "backup" H2 memory database. I make sure that this is done
    once and only once.
 2. Before each test runs, I copy the "backup" database to a
    "primary" database.
 3. After each test runs, I drop the "primary" database.

If you were using MySQL instead of H2, you might use `mysqldump` to
create a snapshot of the "backup" database after step 1,
and restore it to the "primary" database in step 2.

I use JUnit 5 in my test suite, and make use of the extension system.
An example extension might look like:

```java
package com.maxenglander.examples.testwithh2;

import java.sql.DriverManager;
import java.util.concurrent.AtomicBoolean;

import org.junit.jupiter.api.extension.AfterEachCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.BeforeEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;

public class DatabaseExtension implements AfterEachCallback,
        BeforeAllCallback, BeforeEachCallback {
    private static String DB_OPTIONS = "?MODE=MYSQL";
    private static String DB_BACKUP = "memFS:/backup";
    private static String DB_BACKUP_URL = "jdbc:h2:" + DB_BACKUP + DB_OPTIONS;
    private static String DB_PRIMARY = "memFS:/primary";
    private static String DB_PRIMARY_URL = "jdbc:h2:" = DB_PRIMARY + DB_OPTIONS;

    private static AtomicBoolean RAN_MIGRATIONS = new AtomicBoolean();

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        try(Connection connection
                = createConnection(DB_PRIMARY_URL) {
            // This is the H2 equivalent of "drop database primary;"
            connection.prepareStatement("DROP ALL OBJECTS DELETE FILES;");
        }
    }

    @Override
    public synchronized void beforeAll(ExtensionContext context) throws Exception {
        if(!RAN_MIGRATIONS.get()) return;
        try(Connection connection
                = createConnection(DB_BACKUP_URL)) {
            // Run migrations with e.g. Liquibase
        }
        RAN_MIGRATIONS.set(true);
    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        copyDatabase(DB_BACKUP, DB_PRIMARY);
    }

    private void copyDatabase(String srcDb, String dstDb) {
        // getParent("jdbc:h2:memFS:/backup") => "jdbc:nioMemFs:"
        String srcDir = FileUtils.getParent(srcDb);

        // getName("jdbc:h2:memFS:/backup") => "/backup"
        String srcName = FileUtils.getName(srcDb);

        for(String srcFileName
                : FileLister.getDatabaseFiles(srcDir, srcName, false) {
            // E.g.  "jdbc:h2:memFS:/backup.mv.db"
            //           .substring("jdbc:h2:memFS:/backup".length()) => ".mv.db"
            String basename = srcFileName.substring(DB_BACKUP.length());

            // E.g. "jdbc:h2:memFS:/primary" + ".mv.db" => "jdbc:h2:memFS:/primary.mv.db"
            Strint dstFileName = dstDb + basename;

            IOUtils.copyFiles(srcFileName, dstFileName);
        }
    }

    private Connection createConnection(String url) {
        return DriverManager.getConnection(url);
    }
}
```

## Dealing open connections and unsynced data

I ran into a couple of problems using this approach.

The first problem was that the connection pooling library I use with Hibernate
(C3P0) was keeping connections open to H2 at the end of each test. When I
realized this, I tried logging how many connections were open at the end of
each test, and found that they quickly accummulated so that, after a couple
dozen functional tests, I might have dozens of open JDBC connections.

These open connections were preventing my JUnit extension from dropping the
primary database at the end of each test, which meant that database state
was leaking from one test to the next, resulting, in some cases, in unique
key constraint violations.

The second problem I ran into was that, seemingly at random, certain tests
would complain that database tables couldn't be found, or fail to verify
the presence of a database trigger. Presumably this was a result of the
database restore operation not completing prior to running the next test.

To resolve these issues, I force H2 to close open connections before
dropping or restoring the primary database by issuing `SET EXCLUSIVE 2;`
in both the `afterEach` and `beforeEach` callbacks:

```java
    // ...
    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        try(Connection connection
                = createConnection(DB_PRIMARY_URL) {
            // This is the H2 equivalent of "drop database primary;"
            connection.prepareStatement("SET EXLUSIVE 2; DROP ALL OBJECTS DELETE FILES;");
        }
    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        try(Connection connection
                = createConnection(DB_PRIMARY_URL) {
            connection.prepareStatement("SET EXLUSIVE 2;");
            copyDatabase(DB_BACKUP, DB_PRIMARY);
            // This is the H2 equivalent of "drop database primary;"
        }
    }
    // ...
```

## Closing thoughts and links

I initially tried to use the `mem:` database type before I settled on 
using `memFS`. I also tested `nioMemFS`, which works fine.  In spite
of a [claim][1] by one of the H2 maintainers, it seems to me that
whatever code that implements the `mem:` database does not expose a
[`FileChannel`][2] API, which means that its contents cannot be copied
in the same way.

One possible improvement that I'd like to try out would be to
create a unique database for each test, rather than reusing the
same primary database for each test. This could potentially result
in a performance improvement, from avoiding the need to drop the
database and force connections to close at the end of each test.
It would require injecting the unique database name into each test,
which, in JUnit 5, is trivial thanks to the [`ParameterResolver`][2]
API.

 [1]: https://groups.google.com/forum/#!searchin/h2-database/max$20englander%7Csort:date/h2-database/0J5ZHlxVptY/0yvqODvxBQAJ
 [2]: https://docs.oracle.com/javase/7/docs/api/java/nio/channels/FileChannel.html
 [3]: http://junit.org/junit5/docs/current/api/org/junit/jupiter/api/extension/ParameterResolver.html
