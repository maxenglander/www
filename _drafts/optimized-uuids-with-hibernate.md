---
title: Optimizing MySQL UUID-based IDs with JPA 2.1 Hibernate 5.x
layout: post
---

# Optimizing MySQL UUID-based IDs with JPA 2.1 and Hibernate 5.x

For certain use cases, [version-1 UUIDs][1] can be a useful type for
globally-unique IDs within MySQL. However, insertion of version-1 UUIDs into an
index can create performance problems due to their internal structure. In this
post, I want to share how to use Hibernate to implement a common solution for
this problem. Note that this post applies to MySQL 5.x using the InnoDB engine,
version-1 UUIDs, JPA 2.1 and Hibernate 5.x.

## UUIDs: problems and solutions

The details regarding the internal structure of version-1 UUIDs, the performance
problems that ensue for MySQL InnoDB indexes, and the solutions that other
people have proposed are well-covered elsewhere.

So, I will just summarize by saying that, because version-1 UUIDs include a
timestamp in which the hi- and low- digits of the timestamp are swapped, they
are inserted somewhat randomly into an index, resulting in increased page splits,
larger index size, slower insert speeds, and, depending on the query, slower
lookup speed.

The solution, simply put, is to un-swap the hi- and low- digits upon insert

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
to re-arrange the UUID digits with MySQL UDFs, than using `@Converter` is a good
option. However, this comes with a big limitation: it cannot be applied to
primary key columns due to a limitation stated in the [JPA docs][6]:

> The Convert annotation should not be used to specify conversion of the
> following: Id attributes, version attributes, relationship attributes, and
> attributes explicitly denoted as Enumerated or Temporal. Applications that
> specify such conversions will not be portable.

```java
/*
 * Auto apply this converter to all UUID-based attributes, excluding the list 
 * of exceptions mentioned in the JPA spec.
 */
@Converter(autoApply=true)
public class UUIDConverter implements AttributeConverter<UUID, byte[]> {
    @Override
    public byte[] convertToDatabaseColumn(final UUID uuid) {
        final byte[] out = new byte[16];
        final byte[] msbIn = fromLong(uuid.getMostSignificantBits());
        
        // Reorder the UUID, swapping the hi- and low- digits of the timestamp
        System.arraycopy(msbIn, 6, out, 0, 2);
        System.arraycopy(msbIn, 4, out, 2, 2);
        System.arraycopy(msbIn, 0, out, 4, 4);

        // The remainder of the UUID (clock sequence and MAC address) are kept as-is
        System.arraycopy(fromLong(uuid.getLeastSignificantBits()), 0, out, 8, 8);
        
        return out;
    }

    @Override
    public UUID convertToEntityAttribute(final byte[] bytes) {
        byte[] msb = new byte[8];
        byte[] lsb = new byte[8];
        
        // Reorder the UUID, swapping the hi- and low- digits of the timestamp
        System.arraycopy(value, 4, msb, 0, 4);
        System.arraycopy(value, 2, msb, 4, 2);
        System.arraycopy(value, 0, msb, 6, 2);

        // The remainder of the UUID (clock sequence and MAC address) are kept as-is
        System.arraycopy(value, 8, lsb, 0, 8);
        
        return new UUID(asLong(msb), asLong(lsb));
    }
}
```

### Using a Hibernate custom type

Using a Hibernate custom type is conceptually similar to using a `@Converter`,
without the limitations around `@Id` attributes.
    
 [1]: https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_1_.28date-time_and_MAC_address.29
 [2]: https://www.percona.com/blog/2007/03/13/to-uuid-or-not-to-uuid/
 [3]: http://mysql.rjweb.org/doc.php/uuid
 [4]: https://www.percona.com/blog/2014/12/19/store-uuid-optimized-way/
 [5]: https://www.percona.com/blog/2015/04/03/illustrating-primary-key-models-in-innodb-and-their-impact-on-disk-usage/
 [6]: https://docs.oracle.com/javaee/7/api/javax/persistence/Converter.html
