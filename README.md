# A partial implementation of Protocol Buffers in Idris

Protocol buffers are a serializable format for structured data that is
ubiquitous at Google.  Idris is a language where types are first class values
that is useful for theorem proving and metaprogramming.  This package contains a
partial implementation of protocol buffers in the Idris language.  The goal is
to demonstrate how Idris is capable of advanced metaprogramming that simplifies
the implementation of protocol buffers.  

This is not an official Google product.  This project is a personal 20% project.

Protocol buffers are a way to describe a schema for serializable data.  The
schema is called a protocol buffer "message".  A message describes the kind of
data that is to be serialized, and comprises a set of fields.  Each field can be
a primitive type, an enum, or another message.  In this way protocol buffers
allow the description of arbitrary data types.  Below is shown a protocol
message describing a phone number, including an enum definition for the type
of a phone number (mobile, home or work).
```
enum PhoneType {
  MOBILE = 0;
  HOME = 1;
  WORK = 2;
}
message PhoneNumber {
  required string number = 1;
  optional PhoneType type = 2 [default = HOME];
}
message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;
  repeated PhoneNumber phone = 4;
}
```

In most languages, protocol buffers are turned into native data types by code
generation.  E.g. in C++, the generated code for the `Person` class will be
a C++ class with methods like `set_name`, `name` and more complex method to
set nested messages like `phone`.  The code is generated by a compiler that
takes protocol message descriptions like the one above, parses them and emits
generated C++ code.

However in Idris, we can avoid generating code by considering a type of
message descriptors (so the `Person` message description would be an value with
this type).  This is the `MessageDescriptor` class.  In the protocol buffers
documentation this is just called a `Descriptor` but here it's useful to be
more consistent in naming.  The unique property of Idris is that we can
define an inductive data type `InterpMessage` which has `MessageDescriptor` for
its index space, so that `InterpMessage Person` is the type of messages of
type `Person`.  Inductive data types are similar to C++ templates but with the
advantage their arguments can be any value and that value need not even be
known at compile time.

In Idris we can write generic, type safe, code for working with protocol
buffers.  In particular we have code for serialization and deserialization to
and from text form.  Serialization looks like
```
printToTextFormat : InterpMessage d -> String
```
Note that `d` is an implicit argument.  If we were to expand the above
declaration to include the type of `d` (which Idris is able to infer), it would
be
```
printToTextFormat : {d : MessageDescriptor} -> InterpMessage d -> String
```
Note that this function is generic in that `d` ranges over the space of
message descriptors, and for each `d`, the type of the second argument is
a message of type `InterpMessage d`, that is a protocol message in the format
given by the descriptor `d`.  Deserialization can also be done generically.

## About this Repo

The main purpose of the repo is to demonstrate and explore some interesting
applications of Idris.  Most of the interesting applications are in the `Test`
directory which contains an example descriptor `Person` along with some
examples of creating an instance of `InterpMessage Person` and some methods
to inspect properties.

The `Person` descriptor is given by
```
PhoneType : EnumDescriptor
PhoneType = MkEnumDescriptor [
  MkEnumValueDescriptor "MOBILE" 0,
  MkEnumValueDescriptor "HOME" 1,
  MkEnumValueDescriptor "WORK" 5
]

PhoneNumber : MessageDescriptor
PhoneNumber = MkMessageDescriptor [
  MkFieldDescriptor Required PBString "number",
  MkFieldDescriptor Optional (PBEnum PhoneType) "type"
]

Person : MessageDescriptor
Person = MkMessageDescriptor [
  MkFieldDescriptor Required PBString "name",
  MkFieldDescriptor Required PBInt32 "id",
  MkFieldDescriptor Optional PBString "email",
  MkFieldDescriptor Repeated (PBMessage PhoneNumber) "phone"
]
```
A example of an element of `InterpMessage Person` is
```
John : InterpMessage Person
John = MkMessage [
  "John Doe",
  1234,
  Just "jdoe@example.com",
  [
    MkMessage ["123-456-7890", Just 1],
    MkMessage ["987-654-3210", Nothing]
  ]
]
```
Note that the `MkMessage` constructor accepts a heterogeneous list.  This is
not an `HVect` but a data type we construct in `Protobuf.idr` whose constructors
are `Nil` and `(::)`, but whose elements have types corresponding to the
types of the fields given by the message descriptor.

## Installing

### Install the [Lightyear](https://github.com/ziman/lightyear) package.
This can be done by cloning the repo by running this command from your home
directory:
```
git clone https://github.com/ziman/lightyear
```
and installing it with
```
cd lightyear
idris --install lightyear.ipkg
```

### Clone this repo and run the tests.
Clone this repo by running the following from your home directory
```
git clone https://github.com/google/idris-protobuf
```
The `Test` directory contains interesting examples that you might want to modify
and re-run.  To run the tests run this command from this repo
```
cd idris-protobuf
idris --testpkg protobuf.ipkg
```

### Experiment in the REPL.  
In order to load the repl with this package, first install it with the command
```
idris --install protobuf.ipkg
```
then load the Idris REPL with this package along with the test utils, with the
command
```
idris -p lightyear -p protobuf Test/Utils.idr
```
While in the REPL you can explore the package, e.g.
```
:module protobuf
*Test/Utils *Protobuf> printToTextFormat Test.Utils.John
"name: \"John Doe\"\nid: 1234\nemail: \"jdoe@example.com\"\nphone: {\n  number: \"123-456-7890\"\n  type: HOME\n}\nphone: {\n  number: \"987-654-3210\"\n}\n" : String
*Test/Utils *Protobuf> parseFromTextFormat {d=Test.Utils.Person} "name: \"Kester Tong\" id: 1234"
Right (MkMessage ["Kester Tong", 1234, Nothing, []]) : Either String
```
