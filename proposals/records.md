
# Records

This proposal tracks the specification for the C# 9 records feature, as agreed to by the C#
language design team.

The syntax for a record is as follows:

```antlr
record_declaration
    : attributes? class_modifier* 'partial'? 'record' identifier type_parameter_list?
      parameter_list? record_base? type_parameter_constraints_clause* record_body
    ;

record_base
    : ':' class_type argument_list?
    | ':' interface_type_list
    | ':' class_type argument_list? interface_type_list
    ;

record_body
    : '{' class_member_declaration* '}'
    | ';'
    ;
```

Record types are reference types, similar to a class declaration. It is an error for a record to provide
a `record_base` `argument_list` if the `record_declaration` does not contain a `parameter_list`.

## Members of a record type

In addition to the members declared in the record body, a record type has additional synthesized members.
Members are synthesized unless a member with a "matching" signature is declared in the record body or
an accessible concrete non-virtual member with a "matching" signature is inherited.
Two members are considered matching if they have the same
signature or would be considered "hiding" in an inheritance scenario.

The synthesized members are as follows:

### Equality members

Record types produce synthesized implementations of the following methods, where `T` is the
containing type:
```C#
public override int GetHashCode();
public override bool Equals(object other);
public virtual bool Equals(T other);
```
`GetHashCode()` and `Equals(object other)` are overrides of the virtual methods in `System.Object`.
Any methods on intermediate base classes that would hide those methods are ignored when overriding.

Derived record types also override the `Equals(TBase other)` method from each base record type.

The record type synthesizes an implementation of `System.IEquatable<T>` that is implicitly implemented by `Equals(T other)` where `T` is the containing type.
Record types do not synthesize implementations of `System.IEquatable<TBase>` for any base type `TBase`,
even if those interfaces are implemented by the base record types.

The base record class synthesizes an `EqualityContract` property. The property is overridden in
derived record classes. The synthesized implementations return `typeof(T)` where `T` is containing type.
```C#
protected virtual Type EqualityContract { get; }
```

It is an error if the base implementations of any of the overridden members is sealed or non-virtual,
or do not match the expected signature and accessibility.

`Equals(T other)` returns true if and only if each of the following terms are true:
- `other` is not `null`, and
- For each field declared in the record type, the value of
`System.Collections.Generic.EqualityComparer<TN>.Default.Equals(fieldN, other.fieldN)` where `TN` is the field type, and
- If there is a base record type, the value of `base.Equals(other)`; otherwise
the value of `EqualityContract.Equals(other.EqualityContract)`.

The overrides of `Equals(T other)` for the base methods, including `object.Equals(object other)`, perform the equivalent of:
```C#
public override bool Equals(object other) => Equals(other as T);
```

`GetHashCode()` returns the `int` result of a deterministic function taking the following values:
- For each field declared in the record type, the value of
`System.Collections.Generic.EqualityComparer<TN>.Default.GetHashCode(fieldN)` where `TN` is the field type, and
- If there is a base record type, the value of `base.GetHashCode()`; otherwise
the value of `System.Collections.Generic.EqualityComparer<System.Type>.Default.GetHashCode(EqualityContract)`.

### Copy and Clone members

A record type contains two copying members:

* A constructor taking a single argument of the record type. It is referred to as a "copy constructor".
* A synthesized public parameterless virtual instance "clone" method with a compiler-reserved name

The purpose of the copy constructor is to copy the state from the parameter to the new instance being
created. This constructor doesn't run any instance field/property initializers present in the record
declaration. If the constructor is not explicitly declared, a protected constructor will be synthesized
by the compiler.
The first thing the constructor must do, is to call a copy constructor of the base, or a parameter-less
object constructor if the record inherits from object. An error is reported if a user-defined copy
constructor uses an implicit or explicit constructor initializer that doesn't fulfill this requirement.
After a base copy constructor is invoked, a synthesized copy constructor copies values for all instance
fields implicitly or explicitly declared within the record type.

The "clone" method returns the result of a call to a constructor with the same signature as the
copy constructor. The return type of the clone method is the containing type, unless a virtual
clone method is present in the base class. In that case, the return type is the current containing
type if the "covariant returns" feature is supported and the override return type otherwise. The
synthesized clone method is an override of the base type clone method if one exists. An error is
produced if the base type clone method is sealed.

If the containing record is abstract, the synthesized clone method is also abstract.

## Positional record members

In addition to the above members, records with a parameter list ("positional records") synthesize
additional members with the same conditions as the members above.

### Primary Constructor

A record type has a public constructor whose signature corresponds to the value parameters of the
type declaration. This is called the primary constructor for the type, and causes the implicitly
declared default class constructor, if present, to be suppressed. It is an error to have a primary
constructor and a constructor with the same signature already present in the class.

At runtime the primary constructor

1. executes the instance initializers appearing in the class-body

1. invokes the base class constructor with the arguments provided in the `record_base` clause, if present

If a record has a primary constructor, any user-defined constructor, except "copy constructor" must have an
explicit `this` constructor initializer. 

Parameters of the primary constructor as well as members of the record are in scope within the `argument_list`
of the `record_base` clause and within initializers of instance fields or properties. Instance members would
be an error in these locations (similar to how instance members are in scope in regular constructor initializers
today, but an error to use), but the parameters of the primary constructor would be in scope and useable and
would shadow members. Static members would also be useable, similar to how base calls and initializers work in
ordinary constructors today. 

Expression variables declared in the `argument_list` are in scope within the `argument_list`. The same shadowing
rules as within an argument list of a regular constructor initializer apply.

### Properties

For each record parameter of a record type declaration there is a corresponding public property
member whose name and type are taken from the value parameter declaration.

For a record:

* A public `get` and `init` auto-property is created (see separate `init` accessor specification).
  Each "matching" inherited abstract accessor is overridden. The auto-property is initialized to
  the value of the corresponding primary constructor parameter.

### Deconstruct

A positional record synthesizes a public void-returning method called Deconstruct with an out
parameter declaration for each parameter of the primary constructor declaration. Each parameter
of the Deconstruct method has the same type as the corresponding parameter of the primary
constructor declaration. The body of the method assigns each parameter of the Deconstruct method
to the value from an instance member access to a member of the same name.

## `with` expression

A `with` expression is a new expression using the following syntax.

```antlr
with_expression
    : switch_expression
    | switch_expression 'with' '{' member_initializer_list? '}'
    ;

member_initializer_list
    : member_initializer (',' member_initializer)*
    ;

member_initializer
    : identifier '=' expression
    ;
```

A `with` expression allows for "non-destructive mutation", designed to
produce a copy of the receiver expression with modifications in assignments
in the `member_initializer_list`.

A valid `with` expression has a receiver with a non-void type. The receiver type must contain an
accessible synthesized record "clone" method.

On the right hand side of the `with` expression is a `member_initializer_list` with a sequence
of assignments to *identifier*, which must be an accessible instance field or property of the return
type of the `Clone()` method.

Each `member_initializer` is processed the same way as an assignment to a field or property
access of the return value of the record clone method. The clone method is executed only once
and the assignments are processed in lexical order.
