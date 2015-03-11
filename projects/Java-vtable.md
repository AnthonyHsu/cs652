# Translating a subset of Java to C

## Goal

In this project, you must translate a very small subset of Java to pure C using ANTLR and Java code that you write. The subset has very few statements and almost no expressions, focusing instead on classes and methods. You will learn not only about  language translation but also how polymorphism is implemented using so-called `vtables`, which C++ also uses. It requires a deep understanding of C pointer types as well.

To get started, please familiarize yourself with the [Java translator starter kit](https://github.com/USF-CS652-starterkits/parrt-vtable). The main program is `JTran.java`.

## Discussion

### Sample input

To get a flavor of the Java subset, take a look at the following `.j` files, which demonstrate all of the tricky polymorphism and dynamic binding your translated C code must implement.

```java
// tests/cs652/j/polymorph.j
class Animal {
    int ID;
    int getID() { return ID; }
    void speak() { printf("%d\n", getID()); }
}

class Dog extends Animal {
    void speak() { printf("woof!\n"); }
}

Animal a;
Dog d;
d = new Dog();
a = d; // should cast to Animal *
a.speak(); // prints woof!
d.speak(); // prints woof!
```

Check out the [expected C code](https://github.com/USF-CS652-starterkits/parrt-vtable/blob/master/tests/cs652/j/polymorph.c).

```java
// tests/cs652/j/vtable_check.j
class Animal {
    int getID() { return 1; }
    int foo() { return getID(); }
}

class Dog extends Animal {
    int getID() { return 2; }
}

class Pekinese extends Dog {
    int getID() { return 3; }
}

Pekinese d;
d = new Pekinese();
printf("%d\n", d.foo()); // must print 3
```

Check out the [expected C code](https://github.com/USF-CS652-starterkits/parrt-vtable/blob/master/tests/cs652/j/vtable_check.c).

A file consists of zero or more class definitions file optionally by a main program followed by end of file:

```
grammar J;

file:   classDeclaration* main EOF
    ;
```

You can see all of the [sample inputs I used for testing](https://github.com/USF-CS652-starterkits/parrt-vtable/tree/master/tests/cs652/j).

### Translation to C

There are very few expression and statement constructs that you need to translate. This project is all about the translation of classes to `struct`s and methods to functions. One of the trickiest part is converting the flexible Java type system to C's much stricter type system. For example, in Java and animal can point at a dog but not in C. We have to typecast any assignment right hand side and parameter expression.  We also have to deal with function pointers so you should read the following carefully: [How To Read C Declarations](http://blog.parr.us/2014/12/29/how-to-read-c-declarations/).

#### Boilerplate support code

For simplicity, generate all of the support code at the start of each file:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    char *name;
    int size;
    void (*(*_vtable)[])();
} metadata;

typedef struct {
    metadata *clazz;
} object;

object *alloc(metadata *clazz) {
    object *p = malloc(clazz->size);
    p->clazz = clazz;
    return p;
}
```

`metadata` records information about a class definition, including its name, how many bytes are required to hold an  instance of that class, and finally its `vtable`.

Each instance of a class starts with a single pointer of overhead, a pointer to its class definition "object" called `class`. This memory template is described by `object`. All instances that have data fields will be bigger than `object` by using this `struct` allows us to access any objects class definition pointer. To make a method call, we need to access the receiver objects `vtable`.

Finally, we have a function that allocates space for an instance of a class: `alloc`. It takes class definition metadata and returns an object of the appropriate size with its class definition pointer set.  Constructor expressions such as `new Dog()` become calls to `alloc` with a typecast on the result so that it is appropriately typed for C code.

```c
((Dog *)alloc(&Dog_metadata))
```

#### Classes with fields

Classes in J become `struct`s in C and the translation is pretty simple.

```java
class T {
    int x;
    Dog y;
}
```

becomes:

```c
// D e f i n e  C l a s s  T
typedef struct {
    metadata *clazz;
    int x;
    Dog *y;
} T;
```

Don't forget the metadata pointer so that every object knows its type at runtime. Most importantly, it is through this class definition object that we can access the `vtable` to do method calls. (More later on method calls).

Every reference to a field `x` within a method is really `this.x` and so field references inside methods are converted to `this->x` where `this` is the first argument of each function we define for each J method.

Accesses to fields outside are always through an object reference. `o.x` &rarr; `o->x`.

#### Inheritance of fields

C does not support inheritance of `struct`s and so the translation needs to copy fields from all superclasses into subclasses. For example,

<table border="0">
<tr>
<td><pre>
class Animal {
    int ID;
}
class Dog extends Animal {
}
</td>
<td>
<pre>typedef struct {
    metadata *clazz;
    int ID;
} Animal;
typedef struct {
    metadata *clazz;
    int ID;
} Dog;
</td>
</tr>
</table>

#### Main programs

The statements and local variable declarations of a J program translate to the body of a `main` function in C:

<table border="0">
<tr>
<td><pre>int x;
x = 1;
printf("%d\n", x);
</td>
<td>
<pre>int main(int argc, char *argv[])
{
    int x;
    x = 1;
    printf("%d\n", x);
}
</td>
</tr>
</table>

#### Polymorphism

Polymorphism is the ability to have a single pointer refer to multiple types. In Java, references to an identifier of type `T` become pointers to `T` in C: `Dog d;` &rarr; `Dog *d;`. Consider the following J code with a final assignment of a `Dog` to an `Animal`.

<table border="0">
<tr>
<td><pre>
Animal a;
Dog d;
d = new Dog();
a = d; // should cast to Animal *
</td>
<td>
<pre>
Animal *a;
Dog *d;
d = ((Dog *)alloc(&Dog_metadata));
a = ((Animal *)d);
</td>
</tr>
</table>

The C code requires a cast in the assignment, as you can see in the last C statement.  `Animal *` and `Dog *` are never compatible in C and so we need the typecast. We are naturally just doing a pointer assignment but we have to hush the C compiler.

#### Method translation

Methods translate to functions in C. To distinguish between methods with the same name in different classes, we prefix the C function name with the class name: `foo` &rarr; `T_class` for method `foo` in class `T`. Each C function also takes the receiver object as an additional first argument as shown in the following translation.

<table border="0">
<tr>
<td><pre>
class T {
    void foo() { printf("hi\n"); }
}
</td>
<td>
<pre>
typedef struct {
    metadata *clazz;
} T;
void T_foo(T *this)
{
    printf("hi\n");
}
</td>
</tr>
</table>

#### Late binding (dynamic method dispatch)

According to [Wikipedia](http://en.wikipedia.org/wiki/Dynamic_dispatch), "dynamic dispatch is the process of selecting which implementation of a polymorphic operation (method or function) to call at run time." I think of this as sending messages to a receiver object that determines how to respond ala SmallTalk.

Method invocation expressions in Java such as `o.m(args)` are executed as follows:

1.	Ask `o` to return its type (class name); call it `T`.
2.	Load `T.class` if `T` is not already loaded.
3.	Ask `T` to find an implementation for method m. If T does not define an implementation, `T` checks its superclass, and its superclass until an implementation is found.
4.	Invoke method `m` with the argument list, `args`, and also pass `o` to the method, which will become the `this` value for method `m`.

In C++, and in our translation of Java, we will do something very similar but of course we do not need to load `T` dynamically. Lets go through an example to figure out how to implement dynamic binding of methods:

```java
class Vehicle { // implicit extends Object
    void start() { }
    int getColor() { return 9; }
}
class Car extends Vehicle {
    void start() { }
    void setDoors(int n) { }
}
class Truck extends Vehicle {
    void start() { }
    void setPayload(int n) { }
}
```

Both `Car` and `Truck` inherit `getColor`.

If I have a reference to a truck and send it message `start`, we need to figure out whether to execute `Truck`'s `start` or `Vehicle`'s version:

```java
Vehicle v;
Truck t = new Truck();
v = t;
t.start();
v.start(); // same thing as t.start() in this case
```

From above, we know that the `start` methods will be translated to  multiple C functions such as:

```c
void Vehicle_start(Vehicle *this) { }
void Truck_start(Truck *this) { }
```

Given a vehicle reference, `v`, you might be tempted to simply call `Truck_start(v)` but that would be static binding not dynamic binding. We need to do the equivalent of testing the type of object  pointed at by `v` and then call the appropriate function implementation. A more sophisticated way to do that is through a function pointer. In the following possible translation (not the way we will eventually do it), we use a function pointer that points at the appropriate function.

```c
typedef struct {
    metadata *clazz;
    void (*start)(); // set this to &Vehicle_start
    ...
} Vehicle;
typedef struct {
    metadata *clazz;
    void (*start)(); // set this to &Truck_start
    ...
} Truck;
```

It is important that the `start` function pointer sit at the same offset in each struct. Then, the translation of method calls to function calls becomes an indirection through a function pointer:

<table border="0">
<tr>
<td><pre>t.start();
v.start();
</td>
<td>
<pre>(*t->start)(t);
(*v->start)(v);
</td>
</tr>
</table>

![late binding example](images/vtable_example.png)

shown example where the order will not matter.

## Tasks

![data flow](images/vtable-data-flow.png)

### Creating the J grammar

Your must fill in the `J.g4` grammar by looking at all of the examples and the standard [ANTLR Java grammar](https://github.com/antlr/grammars-v4/blob/master/java/Java.g4). (I used as a template to cut it down to my `J.g4`.) Learning how to examine exemplars of a language and construct a suitable grammar is important but here are a few details that matter in terms of symbol table management and type analysis.

* Assume all input is syntactically and semantically valid J(ava) code with the exception that statements existing outside of class definitions are considered the main program. Other than that, assume Java syntax and semantics.
* Support method and field inheritance.
* Support method overriding but not overloading.
* Only integers and floating-point literals are valid. For floating-point numbers, don't worry about negation or exponents: just match and integer on either side of a decimal point.
* Identifiers are just the usual upper and lowercase letters, underscores, and digits (but not in the first position).
* Allow `null` and `this`
* Constructor definitions are not allowed but we still use syntax `new T()` (without parameters) to create objects.
* There are no access modifiers like `public`; everything is assumed to be `public`.
* To print things out there is a single predefined function called `printf(STRING)` or with variable number of arguments `print(STRING, args...)`.
* String literals are only allowed as the first argument of `printf()` calls. Strings should allow a single escape of `\"` but none other.
* There are no variable initializers like `int x=1;`. You must do `int x; x=1;`.
* There are no operators and so you don't have to worry about operator precedence but you do have to support criticize expressions for grouping.
* `void` methods are allowed.
* `this.foo()` and `foo()` methods calls are allowed inside class definitions but not `super`.
* `a.b.c.foo()` style calls are allowed outside of classes and inside methods of classes.
* `t.y` and `this.y` and `y` field access is allowed inside methods and `t.y` is allowed outside of methods. `x` without qualifications is a local variable outside of a method (and could be within a method).

### Defining scopes and symbols


2. Define J symbol table objects using `src/org/antlr/symbols` objects as superclasses as necessary:
	JArg.java
	JClass.java
	JField.java
	JMethod.java
	JObjectType.java
	JPrimitiveType.java
	JVar.java
2. `DefineScopesAndSymbols.java`

### Computing expression types

3. `SetScopes.java`
4. `ComputeTypes.java`

### Constructing a model

![output model objects](images/vtable_models.png)

### Generating C code from the model

## Testing

All [sample inputs I used for testing](https://github.com/USF-CS652-starterkits/parrt-vtable/tree/master/tests/cs652/j) are available. For each test `T`, you will find `T.j`, `T.c`, and `T.txt` where `T.txt` is the expected output after you compile and execute the program. You can run all of the tests like this:

```bash
./bild.py -debug tests
```

Remember, the definition of “working” is when your grammar correctly parses all of the .j files. If you use the -tree option from the command line with JTran.java that I provide, it will pop up a visual of the parse tree for you.