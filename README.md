# Effective Java In a Nutshell
Highlights the key points from the famous Java "Best Practices" book - [Effective Java](https://www.amazon.com/Effective-Java-3rd-Joshua-Bloch/dp/0134685997) by Joshua Bloch

## Table of contents

1. [Creating and destroying objects](#create_destroy)
   1. [Static Factory method vs. Constructor](#create_destroy_sf)
   2. [Builder pattern vs. multiple constructors](#create_destroy_builder)
   
   

<a id='create_destroy' />

## Creating and destroying objects

<a id='create_destroy_sf' />

### Static Factory method vs. Constructor

Try to use static factory methods (SFM) to create an object of a class, instead of exposing multiple constructors with different arguments, created for different purposes. 

#### Advantages

*  **Unlike constructors, SFS's have names** - A constructor cannot clearly explain, what kind of object it creates. Even if we name the constuctor arguments in an intuitive way, that wouldn't help much. SFM's can have **Meaningful** name which can help clients to understand what kind of objects they can expect, when they invoke the method. Example  - `BigInteger.probablePrime(...)`

* **Unlike constructors, SFM's are not required to create NEW object, everytime they are invoked** - We can cache and reuse the objects using SFM's, if the cost of creating those objects are very high. Also, we can do *instance controlling* (maintaining Singleton instance of the class).

* **SFM's can return any sub-type of their return type** - This gives great flexibility. We can create private classes which extend the return type of the SFM's, in turn making overall API compact. Example - `Collections` class (Imagine if all the classes returned by utility functions of `Collections` class were created as `public` classes!

Secondly, a SFM can return multiple type of objects, depending on the parameter value passed to the method. Example - `EnumSet.of(...)` method which returns `RegularEnumSet` or `JumboEnumSet` depending on the size of `enum` declared.

Thirdly, SFM can help to develop *service provider framework* (Java JDBC API).

#### Disadvantages

* **Creating only SFM's without public or protected constructor** makes the class non-extendable.

* **SFM's if not given a correct name, are NOT readily distinguishable from other static methods** - It is important to follow some standard conventions to distinguish the SFM's from regular static methods.

  * `valueOf`— Returns an instance that has, loosely speaking, the same value as its parameters. Such static factories are effectively type-conversion methods.
  * `of` — A concise alternative to valueOf, popularized by `EnumSet`
  * `getInstance` — Returns an instance that is described by the parameters but cannot be said to have the same value. In the case of a singleton, `getInstance` takes no parameters and returns the sole instance.
  * `newInstance`— Like `getInstance`, except that `newInstance` guarantees that each instance returned is distinct from all others.
  * `getType` — Like `getInstance`, but used when the factory method is in a different class. `Type` indicates the type of object returned by the factory method.
  *  `newType` — Like `newInstance`, but used when the factory method is in a different class. `Type` indicates the type of object returned by the factory method.
  

### Builder pattern vs. multiple constructors

Often, we want to create a class with some required values and some optional values. For this reason, we can create **Telescopic Constructors** (a set of constructors, where one of them accepts the required arguments, followed by others which accept - all the required arguments + optional set and so on...). Let's see an example below. To create an "Account", email and password fields are mandatory and rest all are optional.

    class Account {
        
        private final String email; // required
        private final String password; // required
        private final String name; // optional
        private final String address; // optional
        private final int age; // optional
      
        public Account(String email, String password) {
          this(email, password, null);
        }
        
        public Account(String email, String password, String name) {
          this(email, password, name, null);
        }
        
        public Account(String email, String password, String name, String address) {
          this(email, password, name, address, 0);
        }
        
        public Account(String email, String password, String name, String address, int age) {
           this.email = email;
           this.password = password;
           this.name = name;
           this.address = address;
           this.age = age;
        } 
        
        
    }

The disadvantage of above approach is, we are forced to pass the value for a parameter which we don't want to set, if we use "All parameter constructor". Also, as the number of parameters increase, the constructors become unmanageble and create confusion for clients.

One of the possible alternative will be to use **Java Bean** approach, but it is too verbose and it doesn't gurantee the creation of a *consistent* object (client can set only 1 required parameter and can start using the object). Also we need to forego the *immutability*.

#### Builder Pattern

Builder pattern, gives best of both (Telescopic Constructor & Java Bean). Each class will have its own `Builder` class. This will expose constructor to get required parameters from client and setters to set other optional parameters. 


  
    class Account {
        
        private final String email; // required
        private final String password; // required
        private final String name; // optional
        private final String address; // optional
        private final int age; // optional
      
        public static class Builder {
        
          // required
          private final String email; 
          private final String password; 
          
          // optional
          private final String name; 
          private final String address;
          private final int age;
          
          Builder(String email, String password) {
            this.email email;
            this.password = password;
          }
          
          public Builder name(String val) {
            name = val;
            return this;
          }
          
          public Builder address(String val) {
            address = val;
            return this;
          }
          
          public Builder age(int val) {
            age = val;
            return this;
          }
        
          public Account build() {
            return new Account(this);
          }
        }
        
        private Account(Builder b) {
          this.email = b.email;
          this.password = b.password;
          this.name = b.name;
          this.address = b.address;
          this.age = age;
        }
        
    }

Now creating a new `Account` object will be as easy as this - `Account a = new Account.Builder("abc", "test").name("Abc").address("555, test street").age(22).build();`







