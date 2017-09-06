---
title: Optimizing MySQL UUID-based IDs with JPA 2.1 Hibernate 5.x
layout: post
---

<a id="top"></a>

# Optimizing MySQL UUID-based IDs with JPA 2.1 and Hibernate 5.x

For certain use cases, [version-1 UUIDs][1] can be a useful type for
globally-unique IDs within MySQL. However, insertion of version-1 UUIDs into an
index can create performance problems due to their internal structure. In this
post, I want to share how to use Hibernate to implement a common solution for
this problem. Note that this post applies to MySQL 5.x using the InnoDB engine,
version-1 UUIDs, JPA 2.1 and Hibernate 5.x.

## UUIDs: problems and solution

The details regarding the internal structure of version-1 UUIDs, the performance
problems that ensue for MySQL InnoDB indexes, and the solutions that other
people have proposed are well-covered elsewhere (see [References](#references)).

So, I will just summarize by saying that, because version-1 UUIDs include a
timestamp in which the hi- and low- digits of the timestamp are swapped, they
are inserted somewhat randomly into an index, resulting in increased page splits,
larger index size, slower insert speeds, and, depending on the query, slower
lookup speed.

The solution, simply put, is to un-swap the hi- and low- digits upon insert,
and swap them again during read.

## Integrating with JPA/Hibernate

All of the digit-swapping solutions I have found online involve using a
pair of user-defined MySQL functions (such as `uuid_to_bin`, `uuid_from_bin`) to
reorder digits during inserts and reads.

It is possible to implement something like this in Hibernate using
`@ColumnTransformer`, although this has the drawback of decreasing type-safety,
and cannot, as far as I can tell, be auto-applied to every UUID-based attribute.

The equivalent of `uuid_to_bin` and `uuid_from_bin` can be performed in Java
with either JPA's `@Converter` or by creating a custom Hibernate type. It is
possible, with both of these approaches, to auto-apply either the `@Converter` or
custom Hibernate type so that every UUID attribute is automatically re-arranged
upon insert or select. There is a big disadvantage for `@Converter` in that it
cannot be applied to primary key UUIDs.

### Using `@ColumnTransformer`

A good way to implement this with Hibernate would be to use `@ColumnTransformer`
to modify the way a UUID column is read and written, e.g.:

```java
public class MyEntity {
    @Column(name="my_attribute")
    @ColumnTransformer(
        read="uuid_from_bin(my_attribute)",
        write="uuid_to_bin(?)")
    public UUID getMyAttribute() { ... }
}
```

The downside of this approach is that, as far as I can tell, there's no way to
configure Hibernate to apply a `@ColumnTransformer`, in a global way, to every
UUID column. There's also some loss of type-safety due to referencing the column
name inside of the value of the "read" attribute.

### Using `@Converter` (is not possible for a UUID-based primary keys) 

If type-safety and the ability to reorder UUIDs without applying a
`@ColumnTransformer` to every UUID column are a higher priority than the ability
to re-arrange the UUID digits with MySQL UDFs, then using `@Converter` is a good
option. However, this comes with a big limitation: it cannot be applied to
primary key columns according to the [JPA docs][6]:

> The Convert annotation should not be used to specify conversion of the
> following: Id attributes, version attributes, relationship attributes, and
> attributes explicitly denoted as Enumerated or Temporal. Applications that
> specify such conversions will not be portable.

This isn't really acceptable for my use case; it may be for yours.
Here's the code, anyway:

```java
// File: com/maxenglander/example/optimizeduuid/OptimizedUUIDConverter.java
package com.maxenglander.example.optimizeduuid;

import static java.lang.System.arraycopy;
import java.util.UUID;

import static org.hibernate.internal.util.BytesHelper.asLong;
import static org.hibernate.internal.util.BytesHelper.fromLong;

import javax.persistence.AttributeConverter;
import javax.persistence.Converter;

/*
 * Auto apply this converter to all UUID-based attributes, excluding the list 
 * of exceptions mentioned in the JPA spec.
 */
@Converter(autoApply=true)
public class OptimizedUUIDConverter implements AttributeConverter<UUID, byte[]> {
    @Override
    public byte[] convertToDatabaseColumn(final UUID uuid) {
        final byte[] out = new byte[16];
        final byte[] msbIn = fromLong(uuid.getMostSignificantBits());
        
        // Reorder the UUID, swapping the hi- and low- digits of the timestamp
        arraycopy(msbIn, 6, out, 0, 2);
        arraycopy(msbIn, 4, out, 2, 2);
        arraycopy(msbIn, 0, out, 4, 4);

        // The remainder of the UUID (clock sequence and MAC address) are kept as-is
        arraycopy(fromLong(uuid.getLeastSignificantBits()), 0, out, 8, 8);
        
        return out;
    }

    @Override
    public UUID convertToEntityAttribute(final byte[] bytes) {
        byte[] msb = new byte[8];
        byte[] lsb = new byte[8];
        
        // Reorder the UUID, swapping the hi- and low- digits of the timestamp
        arraycopy(value, 4, msb, 0, 4);
        arraycopy(value, 2, msb, 4, 2);
        arraycopy(value, 0, msb, 6, 2);

        // The remainder of the UUID (clock sequence and MAC address) are kept as-is
        arraycopy(value, 8, lsb, 0, 8);
        
        return new UUID(asLong(msb), asLong(lsb));
    }
}
```

### Using a Hibernate custom type

Using a Hibernate custom type is conceptually similar to using a `@Converter`,
without the limitations around `@Id` attributes. There are different ways to
create a custom Hibernate type. The approach below is [very][7] [similar][8] to the code
Hibernate uses to transform UUIDs to binary (and vice-versa).

**1. First, define the custom type.**
```java
// File: com/maxenglander/example/optimizeduuid/OptimizedUUIDType.java
package com.maxenglander.example.optimizeduuid;

import java.util.UUID;

import org.hibernate.type.AbstractSingleColumnStandardBasicType;
import org.hibernate.type.descriptor.sql.BinaryTypeDescriptor;

public class OptimizedUUIDType extends AbstractSingleColumnStandardBasicType<UUID> {
    static final long serialVersionUID = 1L;

    public static OptimizedUUIDType INSTANCE = new OptimizedUUIDType();

    public OptimizedUUIDType() {
        super(BinaryTypeDescriptor.INSTANCE, OptimizedUUIDTypeDescriptor.INSTANCE);
    }

    public String getName() {
        return "uuid-binary-optimized";
    }

    public boolean registerUnderJavaType() {
        return true;
    }
}
```

**2. Next, re-order the digits of the UUID upon read/write.**
```java
// File:  com/maxenglander/example/optimizeduuid/OptimizedUUIDTypeDescriptor.java
package com.maxenglander.example.optimizeduuid;

import static java.lang.System.arraycopy;
import java.util.UUID;

import static org.hibernate.internal.util.BytesHelper.asLong;
import static org.hibernate.internal.util.BytesHelper.fromLong;
import org.hibernate.type.descriptor.WrapperOptions;
import org.hibernate.type.descriptor.java.AbstractTypeDescriptor;
import org.hibernate.type.descriptor.java.UUIDTypeDescriptor.ValueTransformer;

public class OptimizedUUIDTypeDescriptor extends AbstractTypeDescriptor<UUID> {
    static final long serialVersionUID = 1L;

    public static OptimizedUUIDTypeDescriptor INSTANCE = new OptimizedUUIDTypeDescriptor();

    public OptimizedUUIDTypeDescriptor() {
        super(UUID.class);
    }

    @Override
    public UUID fromString(final String value) {
        return UUID.fromString(value);
    }

    @Override
    public String toString(final UUID value) {
        return value.toString();
    }

    @Override
    public <X> X unwrap(final UUID value, final Class<X> type, final WrapperOptions options) {
        if(value == null) return null;

        if(UUID.class.isAssignableFrom(type)) {
            return type.cast(value);
        }

        if(String.class.isAssignableFrom(type)) {
            return type.cast(value.toString());
        }

        if(byte[].class.isAssignableFrom(type)) {
            return type.cast(ToBytesTransformer.INSTANCE.transform(value));
        }

        throw unknownUnwrap(type);
    }

    @Override
    public <X> UUID wrap(final X value, final WrapperOptions options) {
        if(value == null) return null;

        if(UUID.class.isInstance(value)) {
            return UUID.class.cast(value);
        }

        if(String.class.isInstance(value)) {
            return UUID.fromString(String.class.cast(value));
        }

        if(byte[].class.isInstance(value)) {
            return ToBytesTransformer.INSTANCE.parse(value);
        }

        throw unknownWrap(value.getClass());
    }

    public static class ToBytesTransformer implements ValueTransformer {
        public static final ToBytesTransformer INSTANCE = new ToBytesTransformer();

        @Override
        public UUID parse(final Object value) {
            byte[] msb = new byte[8];
            byte[] lsb = new byte[8];

            arraycopy(value, 4, msb, 0, 4);
            arraycopy(value, 2, msb, 4, 2);
            arraycopy(value, 0, msb, 6, 2);
            arraycopy(value, 8, lsb, 0, 8);

            return new UUID(asLong(msb), asLong(lsb));
        }

        @Override
        public byte[] transform(final UUID uuid) {
            final byte[] out = new byte[16];
            final byte[] msbIn = fromLong(uuid.getMostSignificantBits());

            arraycopy(msbIn, 6, out, 0, 2);
            arraycopy(msbIn, 4, out, 2, 2);
            arraycopy(msbIn, 0, out, 4, 4);
            arraycopy(fromLong(uuid.getLeastSignificantBits()), 0, out, 8, 8);

            return out;
        }
    }
}
```

**3. Finally, ensure the custom type is used for all UUID attributes.**
```java
// File: com/maxenglander/example/optimizeduuid/package-info.java
@TypeDef(defaultForType=UUID.class, typeClass=UUIDOptimizedType.class)
package com.maxenglander.example.optimizeduuid;

import java.util.UUID;

import org.hibernate.annotations.TypeDef;
```

<a id="references"></a>
## References and links

 1. [Hibernate source code: org.hibernate.type.UUIDBinaryType][7]
 1. [Hibernate source code: org.hibernate.type.descriptor.java.UUIDTypeDescriptor][8]
 1. [Percona: to UUID or not to UUID][2]
 1. [Percona: Store UUID optimized way][4]
 1. [Percona: Illustrating primary key models in innodb and their impact on disk usage][5]
 1. [Rick James on MySQL: UUID][3]
 1. [Oracle JavaDocs: javax.persistence.Converter][6]
 1. [Wikipedia: Version 1 UUIDs][1]

 [1]: https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_1_.28date-time_and_MAC_address.29
 [2]: https://www.percona.com/blog/2007/03/13/to-uuid-or-not-to-uuid/
 [3]: http://mysql.rjweb.org/doc.php/uuid
 [4]: https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/
 [5]: https://www.percona.com/blog/2015/04/03/illustrating-primary-key-models-in-innodb-and-their-impact-on-disk-usage/
 [6]: https://docs.oracle.com/javaee/7/api/javax/persistence/Converter.html
 [7]: https://github.com/hibernate/hibernate-orm/blob/5.1/hibernate-core/src/main/java/org/hibernate/type/UUIDBinaryType.java
 [8]: https://github.com/hibernate/hibernate-orm/blob/5.1/hibernate-core/src/main/java/org/hibernate/type/descriptor/java/UUIDTypeDescriptor.java

[Back to top.](#top)
