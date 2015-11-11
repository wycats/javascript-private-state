# Private State for ECMAScript Objects

Stage 0 Proposal

Champions:

    Allen Wirfs-Brock and Yehuda Katz

# Overview

## Some Requirements

- Provide mutable private state for userland objects
  - Supports allocation as a contiguous storage block when object is created
  - Supports inheritance
  - Subclass instances include private state defined by superclasses
- Secure: Only accessible via code using syntactic special forms
  - Not accessible via external reflection API
  - Not reified via ES MOP
- Symmetrical access and declaration
- Private and protected style access controls
- Should not be less ergonomic to use than declared public fields
- Can be decorated

## Nice To Haves

- Should be significantly more ergonomic to use than declared public fields
- Private state for object literals
- "Friend" access controls
- Static private state
- Private helper functions

## Approach

- Private state modeled after ES2015 Internal Slots
- Allocated when object is created and initially set to `undefined`
  - Constructors need to explicitly initialize slots to other values
- Private slots are not properties and have their own distinct access syntax and semantics

## Examples

### Single Private Slot

```js
class {
  private #data1;   // data1 is the name of a private data slot
                    // the scope of 'data1' is the body of the class definition 
  constructor(d) {
    // #data1 has the value undefined here

    // # is used to access a private data slot
    // #data1 is shorthand for this.#data1
    #data1 = d; 
  }

  // a method that accesses a private data slot
  get data() {
    return #data1;
  }
}
```

A `private` declaration within a class body defines a private data slot and associates a name that can be used to access the slot.  Each instance of the class will have a distinct corresponding private data slot that is created and initialized to `undefined` when the object is created.

### Referencing an Undeclared Slot

Within a class definition that lacks an `extends` clause it is a syntax error to try to access a private slot name that has not been explicitly declared within that class definition.

```js
class {
  constructor(d) {
    #data2 = d;
    // ***^ Syntax Error: 'data2' is not a private slot name
  }
}
```

A different rule for class definition that have an `extends` clause will be described in a later section.

### Private Slots Are Lexical and "Class Private"

The code within a class body is not restricted to referencing private slots of the `this` object. The private slots of any instance of the class may be referenced.

```js
class DataObj {
  private #data1;

  constructor(d) {
    #data1 = d;
  }

  // 'another' should be an instance of DataObj
  sameData(another) {
    return #data1 === another.#data1
  }
};

let obj1= new DataObj(1);
let obj2 = new DataObj(2);

console.log( obj1.sameData(obj2) );            // false
consloe.log( obj1.sameData(new DataObj(1)) );  // true
```

The code within static methods may reference the private slots of instance objects.

```js
class DataObj {
  private #data1;

  constructor(d) {
    #data1 = d;
  }

  // 'arg1' and 'arg2' must be an instance of DataObj
  static sameData(arg1, arg2) {  
    return arg1.#data1 === arg2.#data1
  }
};

let obj1 = new DataObj(1);
let obj2 = new DataObj(2);

console.log( DataObj.sameData(obj1,obj2) );             // false
consloe.log( DataObj.sameData(obj1, new DataObj(1)) );  // true
```

### Runtime Errors

It is a run-time error to reference non-existent or inaccessible private slot.

```js
// assuming the preceding definition of DataObj
let obj3 = { data1: 2 };
console.log(DataObj.sameData(obj1, obj3)); // throws a ReferenceError exception
```

This example throws on the access `arg2.#data1` because obj3 does not a private slot `#data1`.  Instead it has a property named `"data1"`.  Private slots are not properties.  A `obj.#data1` private slot access does not access a property named `"data1"` and a `obj.data1` or `obj["data1"]` property access will not access a private slot named `#data1`.

### Private Names are Lexical

They are inaccessible outside of their defining class body.

```js
// Assuming the preceding definition of DataObj and that
// the following code is not within the body of `class DataObj`

let obj4 = new DataObj(4);

// either early Syntax Error or runtime ReferenceError
// depending upon referencing context
console.log(obj4.#data1);
```

### Not On The Prototype

Private slots are not accessible via the prototype chain.

```js
class DataObj {
  private #data1;

  constructor(d) {
    #data1 = d;
  }

  static testProtoAccess(proto) {
    // private slot on proto is directly accessible
    console.log(proto.#data1);

    let child = Object.create(proto);

    // but cannot be indirectly accessed via prototype chain
    console.log(child.#data1);
  }
}

//logs 42 and then throws ReferenceError
DataObj.testProtoAccess(new DataObj(42));
```

### Not Visible to Nested Classes

Private slots names are only visible to the direct class they were declared inside of. They are not visible to nested class definitions.

```js
class DataObj {
  private #data1;

  constructor(d) {
    #data1 = d;
  }

  static testNestedAccess(pDO) {
    // private slot is directly accessible from methods
    console.log(pDO.#data1);

    function fGetData1(aDO) {
       return aDO.#data1;
    }

    // private slot is directly accessible from inner functions
    console.log(fGetData1(pDO));

    class CGetData {
      static getData1() {
        // pDO is visible to inner class but #data1 is not
        return pDO.#data1;
      }
    }

    // try nested class access to outer class private slot
    console.log(CGetData1.getData1());
  }
}

// Throws ReferenceError during definition of
// nested class CGetData
DataObj.testNestedAccess(new DataObj(42));
```

### Not Polymorphic

Private slots names are not polymorphic across different classes.

```js
class DataObj {
  // private declaration 1 (PD1)
  private #data1;

  constructor(d) {
    // reference using PD1
    #data1 = d;
  }

  static sameData(arg1, arg2) {
    // references using PD1
    return arg1.#data1 === arg2.#data1;
  }
};

class NotDataObj {
  // private declaration 2 (PD2)
  private #data1;

  constructor(d) {
    // reference using PD2
    #data1 = d;
  }
};

let obj1= new DataObj(1);
let obj2 = new NotDataObj(1);

// throws Reference Error
console.log( DataObj.sameData(obj1,obj2) );

// Because obj2's private slot is defined by PD2
// but referenced using PD1
```

Private slot access resolution is not solely based upon the *IdentiferName* given to the slot. Instead, each slot is identified by a pair consisting of the *IdentifierName* and a specific `private` declaration of that *IdentifierName* . A reference to a private slot such as `obj.#name` is only valid if `obj` has a private slot named `name` and the same `private` declaration for `name` is in scope for both the definition of the slot and the reference to the slot.

#### Installed on Subclasses

Private slot **storage** is inherited, but access is lexical: subclasses cannot access private slots installed by the superclass (but see the discussion below of protected slots).

```js
class SuperClass {
  // private slot defined in a superclass
  private #data1;
  constructor(d) {
    #data1 = d;
  }

  get data() {
    return #data1
  }
}

class SubClass extends SuperClass {
  private #data2;

  constructor(d1,d2) {
    super(d1)
    #data2 = d2;
  }

  get data2() {
    return #data2
  }
}

let subObj = new SubClass(42, 24);

// inherited method can access inherited slot from subclass instance
console.log( subObj.getData() ); // logs 42

//subclass method can access subclass defined rivate slot
console.log( subObj.getData2() ); //logs 24
```

Subclass instances are created within their locally defined private slots and with the private slots defined by all of the superclasses of the subclass.  However, the inherited private slots are not directly accessible by code ithin the body of the subclass definition.

```js
class BadSubClass extends SuperClass { //runtime Reference Error
  private #data2;

  constructor(d1,d2) {
    super();

    #data1 = d1;
    // ***^ Will cause runtime Reference Error
    // during class definition because 'data1'
    // is not a private slot name of BadSubClass
    #data2 = d2;
  }
}
```

#### Private Names Are Lexically Distinct

Subclass can reuse private slot names used by superclasses.

```js
class ReuseSlotNameSubClass extends SuperClass {
  // a new private slot
  private #data1;

  constructor(d1,d2) {
    super(d1);
    #data1 = d2;
  }

  get data2() {
    return #data1
  }
}

let obj = new ReuseSlotNameSubClass(42, 24);

// inherited method accesses inherited slot named `data1`
console.log( obj.getData() ); // logs 42

// subclass method accesses distinct subclass slot `data1`
console.log( obj.getData2() ); // logs 24
```

Instances of `ReuseSlotNameSubClass` are created with two private slots, each named `#data1`.  However, each slot is associated with a distinct `private` declaration.  An `obj.#data1` access chooses one of the two slot slots based upon which `private` declaration is statically visible at the point of access.

**Rationale:**

- A subclass should not need to be aware of the inaccessible private slot names used by its superclasses.
- Introducing a new private slot name within a superclass should not break already existing subclass definitions that extend the superclass.

#### Protected Slot Definition and Access

A protected data slot is a private slot that *may* be accessed from code within the bodies of subclasses of the class that defined the private slot.

```js
class Base {
  private #slot1;
  protected #slot2;

  constructor (s1,s2) {
    #slot1 = s1;
    #slot2 = s2;
  }
 }
```

Within its defining class definition, a `protected` declaration for a data slot is treated just like a `private declaration`.  However, declaring  a data slot using `protected` makes it available for access from derived subclasses. All of the `protected` date slot names defined by a superclass are automatically included  in the scope of each of its subclasses unless the subclass explicitly includes a `private` or `protected` declaration for the name:

```js
// see above definition of Base
class Derived extends Base {
  getData2() {
    // protected #slot2 access inherited from Base  
    return #slot2;  
  }
}

// will produce runtime ReferenceError
class Derived2 extends Base {
  getData1() {
    // slot1 defined as private rather than protected in Base
    return #slot1;
  }
}

class Derived3 extends Base {
  // adds an additional private slot that hides inherited slot2
  private #slot2;

  getData1() {
    // returns undefined since subclass slot2 was not initialized
    return #slot2;
  }
}
```

When a class definition with an `extends` clause is evaluated all private slot names referenced from within the scope of the class body are checked against the local `private` and `protected` declarations of the class body and the protected slot names provided by the class that is obtained by evaluating  the `extends` class. A runtime `ReferenceError` occurs during class definition if any referenced slot name is neither locally defined nor provided by the `extends` clause.

## Semantics Sketch

### Slot Keys

Slot keys are are internally used to reference a data slot.  Conceptually a slot key consists of a reference to the class that declares the slot and the declared name of the slot.  There are many ways that an implementation might actually represent a slot key.  For example, it might internally assign a symbol value to each unique slot key.

**Issue** Should slot keys be site specific or instance specific?

### Class constructor function object extensions

- Each function object that is a class constructor has an additional interal slot named `[[instanceSlots]]`
  - The value of the `[[instanceSlots]]` internal slot is an ordered List of all instance data slot keys, including inherited slots.
  - The `[[instanceSlots]]` List of a subclass constructor is a new List consisting of the the slot keys for all private and protected data slots declared by the subclass appended to the elements of its superclass' `[[instanceSlots]]` List.
  - The size of a constructor's `[[instanceSlots]]` List is the number of data slots that need to be allocated when an instance of that constructor is created.
- Each class constructor has an additional internal slot named `[[protectedSlotMap]]`
- The value of the internal slot is a List of string->slot key pairs, mapping *IdentifierNames* to data slot keys of inheritable “protected” data slots.  The List includes entires for protected slot names that are inherited from superclasses.
- A subclass adds to its lexical slot bindings the binding pairs from its superclass' `[[protectedSlotMap]]`.
  - But it excludes any binding pairs whose *IdentifierName* is redeclared by a `private` or `protected` declaration within the subclass body.

###Slot Access Semantics:  *MemberExpression* .# *IdentifierName*

- GetPrivate(obj,slot key) and SetPrivate(obj, slotkey, value) are new abstraction operations for accessing private data slots.
  - they are not available via any reflection API
- Parsing maps *IdentiferName* to a slot key using the class definition specific slot map.
- Creates a Data Slot Reference whose base is the value of *MemberExpresion* and whose referenced name is the slot key.
  - “Data Slot Reference” is a new kind of Reference Value
  - GetValue(Ref) for Data Slot References returns GetPrivate(GetBase(Ref), GetReferencedName(Ref))
  - PutValue(Ref, W) for Data Slot Reference returns SetPrivate(GetBase(Ref), GetReferencedName(Ref), W)

## Possible Design Extensions

The following features are not part of the core proposal. They are possible extensions that show how the "nice to have" requirements could be addressed and/or show how this proposal could integrate with other pending proposals.

#### Private Data Slot Initializers

In a manner similar to the [Class Properties Proposal](https://github.com/jeffmo/es-class-static-properties-and-fields), initialzers could be added to the syntax of `private`/`protected` slot declarations.  For example:

```js
class Base {
  private #slot1 = 42;
  protected #slot2 = null;
}
```

The [issues](https://github.com/tc39/tc39-notes/blob/master/es7/2015-09/sept-22.md#54-updates-on-class-properties-proposal) related to evaluation time and ordering of such initializers are significant and essentially the same as for Class Property Initializers.  If both features are adopted then the handling of initializers should be consistent between them. 

### Private Slots in Object Literals

Allow object literals to define private slots. For example: 

```js
let obj = {
   private #data,
   get data() { return this.#data; },
   set data(v) { this.#data=v; }
};
```

Issues:

- `protected` probably doesn't make sense since object literals don't really have a way to statically define their "inheritance".

### Static Slots in a Class

Static data slots are private data slots of class constructor function objects. The slot declarations would be prefixed with the `static` keyword.

Class constructors are similar to object literal values in that they are essentially singleton instances. For this reason,  the semantics of static slot inheritance has issues and solutions similar to those that arise for object literals.

### Per-Class Lexical Scope

Private data slots provide per-instance private state but they don't provide any support for encapsulated procedural decomposition of  methods.  The latter could be accomplished by allowing *FunctionDeclaration* and *GeneratorDeclaration* to occur as a *ClassElement*.  

For example:

```js
class Example {
   private #slot1, #slot2;
   function helper(obj) {return obj.#slot1+obj.#slot2};
   method1() {return helper(this)};
   method2() {return helper(this)*2};
   ...
}
```

Note that class body level functions are only visible within the class body.  They are not visible to subclasses.  However, a base class could use a protected slot (or a class static data slot, if they exist) when it needs to  make such helper functions available to its subclasses.

This features is essentially the same as part of the [Defensible Classes](http://wiki.ecmascript.org/doku.php?id=strawman:defensible_classes) Stage 0 proposal.

### "Friend" Access

In some situations it is useful to allow two or more  classes that are not related via inheritance to accesss the internal state of each other's instances. This  could be accomplished by allowing Symbols to be used as slot keys. For example:

```js
const sharedSecret = Symbol();

class Friendly {
   //this class has a slot that it exposes to its friends
   private #data[sharedSecret];  //defines a slot that has a Symbol as its slot key
   constructor (v) {
      this.#data = v;
   }
}

class Friend1 {
   //this class has access to the data slot of Friendly instances
   
   //allows us to say obj.#theirData
   friend #theirData[sharedSecret]; 
   
   //access uses the Symbol sharedSecret as the slot key
   reportOn(aFriend) {return aFriend.#theirData}; 
  }
}

let f1 = new Friendly(42);
let f2 = new Friend1();
console.log(f2.reportOn(f1);
```

In the above example, `friend` is a contextual keyword when appearing as the first element of a *ClassElement* and preceding an *Identifier*, similar to the handling of `get` and `set`.

Using this scheme, block scoping and explicit parameterization can be used to to manage and constrain friend-style access to private data slots.

This is a particularly speculative idea.
