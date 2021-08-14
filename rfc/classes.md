# Corinna Classes

As per [the grammar](grammar.md), the smallest possible class is `class A
{}` and you could instantiate with `my $object = A->new`. Not very useful, but
it's there. Note that you do not need to specify a constructor.

Here's a vaugely more interesting class. `has` declares a slot (data) and the
`/:\w+/` attributes provide additional behavior. See
[attributes](attributes.md) for more information.

```perl
class Person {
    has $name  :param;              # must be passed to customer (:param)
    has $title :param = undef;      # optionally passed to constructor (:param, but with default)

    method name () {                # instance method
        return defined $title ? "$title $name" : $name;
    }
}
my $villain = Person->new( title => 'Dr.', name => 'Zacharary Smith' );
my $boy     = Person->new( name => 'Will Robinson' );

say $villian->name;   # Dr. Zacharary Smith
say $boy->name;       # Will Robinson
```

In the above, we have a `Person` class with a required name and an optional
title. The constructor is `new` and accepts an even-sized list of key/value
pairs. Perl the [class construction specification](class-construction.md),
duplicate keys to the constructor are not allowed. Nor is a hash reference. If
you wish to provide an an alternate set of arguments to the constructor, write
an alternate constructor (the behavior replaces Moo/se `BUILDARGS).

Note that the `$name` and `$title` attributes are completely encapsulated. If
you want to expose them to the outside world, you would use the `:reader` and
`:writer` attributes.

```perl
common method from_name ($name) {    # 'common method' is a class method
    $class->new( name => $name );    # all methods have $class injected.
}
```

Let's make the class more interesting.

```perl
class Person {
    use DateTime;
    has $name  :param;                    # must be passed to customer (:param)
    has $title :param = undef;            # optionally passed to constructor (:param, but with default)
    has $created      = DateTime->now;    # cannot be passed to constructor (no :param)
    common has $num_people :reader = 0;   # class attribute, defaults to 0 (common, with reader method)

    ADJUST   { $num_people++ }            # called after new(), but before returned to consumer (BUILD)
    DESTRUCT { $num_people-- }            # destructor

    method name () {
        return defined $title ? "$title $name" : $name;
    }
}

say Person->num_people;     # 0
my $villain = Person->new( title => 'Dr.', name => 'Zacharary Smith' );
say $villain->name;         # Dr. Zacharary Smith
say Person->num_people;     # 1
say $villain->num_people;   # 1 class methods can be called on instances
                            # but instance methods can't be called from classnames or methods
my $boy = Person->new( name => 'Will Robinson' );
say $boy->name;             # Will Robinson
say Person->num_people;     # 2
undef $villain;
say Person->num_people;     # 1
undef $boy;
say Person->num_people;     # 0
```

## Versions

Versions require a leaving `v` followed by the `major.minor.patch` numbers from [semantic versioning](https://semver.org/).

```perl
class MyClass v0.1.3 {
    ...
}
```

## Inheritance

Corinna supports single inheritance via the `isa` keyword.

```perl
class Customer isa Person v0.1.0 {
    has $customer_id :param;

    overrides method name () {
        my $name = $self->next::method;
        $name .= " ($customer_id)";
        return $name;
    }
}

say Customer->num_people; # 0 (inherited)
my $customer = Customer->new( name => 'Ford Prefect', customer_id => 42 );
say Customer->num_people; # 1
say $customer->name;      # Ford Prefect (#42)
```

Let's look at the method a bit more closely.

```
01:    overrides method name () {
02:        my $name = $self->next::method;
03:        $name .= " ($customer_id)";
04:        return $name;
05:    }
```

On line 1, the `overrides` tells Corinna we're overriding a parent method. This should:

* Suppress overridden warnings when overriding a parent method
* Generate a compile-time error if there is no parent method

The compile-time error is to prevent cases like the method above, where
`$self->next::method` would ordinarily generate a runtime exception because the
method does not exist.

On line 2, we see two interesting things. First is the `$self` variable automatically
injected into the body of the method. There is also a `$class` variable availabe, but
it's not shown in this example.

Next is the `next::method` part. Usually we see that as part of the C3 mro. It's here because of the
[SUPER-bug in Perl](http://modernperlbooks.com/mt/2009/09/when-super-isnt.html).

## Subroutines versus Methods

In Corinna, methods and subs are not the same thing. Here's a silly example.

```perl
class Iterator::Number {
    use List::Util 'sum';
    has $i = 0;
    has @numbers;   # no attributes allowed on non-scalars

    method push ($num)    { push @numbers => $num    }
    method pop            { pop  @numbers            }
    method shift ()       { return shift @numbers    }
    method unshift ($num) { unshift @numbers => $num }
    method sum ()         { return sum(@numbers)     } # !!!
    method reset ()       { $i = 0                   }
    method exhausted ()   { return $i > $#numbers    }
    method next () {
        if ( not $self->exhausted ) {
            return $numbers[$i];
            $i++;
        }
    }
}

my $iter = Iterator::Number->new;
$iter->push($_) for 1,2,3;
say $iter->sum; # 6
```

In Corinna, methods and subroutines are not the same thing. If we did _not_
have a `sum` method in the above code, attempting to call `$iter->sum` (or
`$self->sum` internally) would generate a 'method not found' error.