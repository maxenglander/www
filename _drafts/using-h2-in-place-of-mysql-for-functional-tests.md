---
title: Testing with H2 instead of MySQL in a Hibernate application
layout: post
---

# Testing with H2 instead of MySQL in a Hibernate application

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

However, let's pretend both you and I you good reasons for using H2, and
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
memory-based database types, such as `nioMemFS`, which will be useful
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

## Seeding H2 with a schema

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

If you were using MySQL instead of H2, you might accomplish this
by creating a snapshot of the "backup" database `mysqldump` after step 1,
and restoring it to the "primary" database in step 2.

I use JUnit 5 in my test suite, and make use of the extension system.
An example extension might look like:

```java
package com.maxenglander.examples.testwithh2;

import java.sql.DriverManager;
import java.util.concurrent.AtomicBoolean;

import org.junit.jupiter.api.extension.AfterEachCallback;
import org.junit.jupiter.api.extension.BeforeAllCallback;
import org.junit.jupiter.api.extension.BeforeEachCallback;

public class DatabaseExtension implements AfterEachCallback,
        BeforeAllCallback, BeforeEachCallback {
    private static String DB_OPTIONS = "?MODE=MYSQL";
    private static String DB_BACKUP = "nioMemFS:/backup";
    private static String DB_BACKUP_URL = "jdbc:h2:" + DB_BACKUP + DB_OPTIONS;
    private static String DB_PRIMARY = "nioMemFS:/primary";
    private static String DB_PRIMARY_URL = "jdbc:h2:" = DB_PRIMARY + DB_OPTIONS;

    private static AtomicBoolean RAN_MIGRATIONS = new AtomicBoolean();

    @Override
    public void afterEach() throws Exception {
        try(Connection connection
                = createConnection(DB_PRIMARY_URL) {
            // This is the H2 equivalent of "drop database primary;"
            connection.prepareStatement("DROP ALL OBJECTS DELETE FILES;");
        }
    }

    @Override
    public synchronized void beforeAll() throws Exception {
        if(!RAN_MIGRATIONS.get()) return;
        try(Connection connection
                = createConnection(DB_BACKUP_URL)) {
            // Run migrations with e.g. Liquibase
        }
        RAN_MIGRATIONS.set(true);
    }

    @Override
    public void beforeEach() throws Exception {
        copyDatabase(DB_BACKUP, DB_PRIMARY);
    }

    private void copyDatabase(String srcDb, String dstDb) {
        // getParent("jdbc:h2:nioMemFS:/backup") => "jdbc:nioMemFs:"
        String srcDir = FileUtils.getParent(srcDb);

        // getName("jdbc:h2:nioMemFS:/backup") => "/backup"
        String srcName = FileUtils.getName(srcDb);

        for(String srcFileName
                : FileLister.getDatabaseFiles(srcDir, srcName, false) {
            // E.g.  "jdbc:h2:nioMemFS:/backup.mv.db"
            //           .substring("jdbc:h2:nioMemFS:/backup".length()) => ".mv.db"
            String basename = srcFileName.substring(DB_BACKUP.length());

            // E.g. "jdbc:h2:nioMemFS:/primary" + ".mv.db" => "jdbc:h2:nioMemFS:/primary.mv.db"
            Strint dstFileName = dstDb + basename;

            IOUtils.copyFiles(srcFileName, dstFileName);
        }
    }

    private Connection createConnection(String url) {
        return DriverManager.getConnection(url);
    }
}
```
