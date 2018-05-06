# Effective Java In a Nutshell
Highlights the key points from the famous Java "Best Practices" book - [Effective Java](https://www.amazon.com/Effective-Java-3rd-Joshua-Bloch/dp/0134685997) by Joshua Bloch

## Table of contents

1. [Creating and destroying objects](#create_destroy)
   1. [Static Factory method vs. Constructor](#create_destroy_sf)
   2. [Builder pattern vs. multiple constructors](#create_destroy_builder)
   3. [Singleton pattern](#create_destroy_singleton)
   4. [Enforce Non-instantiability with private constructor](#create_destroy_non_instantiability)
   5. [Avoid creating unnecessary objects](#create_destroy_avoid_unnecessary)
   6. [Eliminate obsolete references](#create_destroy_obsolete_ref)
   7. [Avoid finalizers](#create_destroy_finalizers)
   

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
  
<a id='create_destroy_builder' />

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


<a id='create_destroy_singleton' />

### Singleton Pattern

**Before Java 1.5** - There were 2 ways of creating singleton class

* `public static final` field - 


      class Singleton {
         public static final INSTANCE = new Singleton();
         
         private Singleton() {}
         
         //.. rest of the methods
      }
      
 * `public static` factory method - 
 
       class Singleton {
         private static final INSTANCE = new Singleton();
         
         private Singleton() {}
         
         public static getInstance() {
            return INSTANCE;
         }
       }
 
Both the above approaches are similar interms of performance. The latter approach gives flexibility of changing the *Singleton* behavior of the class, and can enable the developer to change the implementation to return `Singleton` instance per thread, without changing any of the method declaration. However, both these approaches have some caveats.

* `private` constructor of the class can be invoked using **Reflection**, and it needs to be explicity protected.
*  There is no inbuilt support for **Serialization**. If you declare the class as `Serializable` then, you must also implement the `readResolve()` method to return the same instance, also making sure all the fields of the class as `transient`.

**Post Java 1.5** - A new type has been introcuded called - `enum` types, which gives ironclad gurantee of *Singleton* behavior for the objects of this type. Along with that, `enum` types have inbuilt support for Serialization and protection from reflexive attacks.

Example - 

      enum Singleton {
         INSTANCE;
         
         //.. rest of the methods
      
      }


<a id='create_destroy_non_instantiability' />

### Enforce Non-instantiability with private constructor

If a class merely contains a set of related static methods (***Utility class***), they are not designated to be instantiated. Since Java compiler inserts a no-arg public constructor into a class, if it doesn't explicitly define a constuctor, clients of such utility classes can accidently create an instance, which is not desirable. To avoid that, we should declare a `private` **no-arg constructor** within our utility calss. Example - `Arrays` , `Collections` etc.

We could also have declared such an utility class as `abstract`, which would make it non-instantiable. But there are 2 issues

* Another class can extend our `abstract` utility class and then that sub-type can be instantiated.
* It gives a wrong notion to the client of the utility class, that it is designed to be inherited.

**Implementation**

      class Utility {
            
            private Utility() {
               throw new AssertionError(); //to avoid accidental instantiation within this class.
            }
      
      }

<a id='create_destroy_avoid_unnecessary' />

### Avoid creating unnecessary objects

We should avoid creating unnecessary objects, when there are options to reuse the same.

* **Don't create new String instance** - `String s = new String("abc")` - which will create a new string object on heap, instead of **String pool**. Instead create string as *literals* - `String s = "abc"`, which are guaranteed to be maintained as single copy, throughout a JVM.

* **Reuse the immutable objects** - Use *static factory methods* provided by immutable objects, than using their constructor. For example - `Boolean.valueOf(String)` is preferable than using `new Boolean(String)`. The former, reuses the cached object, since `Boolean` is immutable (same goes with `Integer`, `Long` etc).

* **Reuse in mutable objects too** - Suppose we use some helper objects to compute something, we can declare them as `final static` and instantiate them within the `static` block of the class, instead of creating a new instance of them, everytime we use them.

Let's say, we want to check if a person is *Baby Boomer*, that is, whether the person is born between year 1946 to 1965.

Wrong example - 

         class Person {
               private Date birthDate;

               // DON'T DO THIS!
               public boolean isBabyBoomer() {
                  // Unnecessary allocation of expensive object
                  Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
                  gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0, 0);
                  Date boomStart = gmtCal.getTime();
                  gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0, 0);
                  Date boomEnd = gmtCal.getTime();
                  return birthDate.compareTo(boomStart) >= 0 && birthDate.compareTo(boomEnd) < 0;
               }
         }

Right example - 

         class Person {
               private Date birthDate;

               private static final Date boomStart;
               private static final Date boomEnd;

               static {
                     Calendar gmtCal = Calendar.getInstance(TimeZone.getTimeZone("GMT"));
                     gmtCal.set(1946, Calendar.JANUARY, 1, 0, 0, 0);
                     boomStart = gmtCal.getTime();
                     gmtCal.set(1965, Calendar.JANUARY, 1, 0, 0, 0);
                     boomEnd = gmtCal.getTime();
               }

               public boolean isBabyBoomer() {
                  return birthDate.compareTo(boomStart) >= 0 && birthDate.compareTo(boomEnd) < 0;
               }
         }

* **Be careful with AutoBoxing** - Since Java 1.5 and introduction of *auto boxing*, we can intermix the **primitive** type variables with their respective **wrapper** types. If we do that unintentionally, then we are in for some serious performance issues! For example - 
      
      Long sum = 0;
      for(int i=0; i<Integer.MAX_VALUE; i++) {
         sum += i;
      }
      
Will create `2^32` `Long` objects and changing `Long sum` to `long sum`, will improve the performance by 250 factors (!!!), by avoiding the **creation of new Long objects** and yeild the same result.

<a id='create_destroy_obsolete_ref' />

### Eliminate Obsolete References

Even though Java has *powerful garbage collection (GC)* mechanism, not all objects created by the programs are reclaimed, We need to be careful in certain situations

* **While Managing memory on our own** - If we decide to implement our own `Stack`, which is backed by an array, and if that array holds some elements which are not of any use (Elements which are popped out of the Stack), we should explicitly set them to null, so that GC becomes aware of it. Unless we do that, GC will refrain from reclaiming that *Unused* object, since we have a strong reference to the whole array!

Bad implementation - 

      public E pop() {
         //BAD implementation!, the object previously referred by 'size' index is still present on the 'elements' array!
         return elements[size--];
      }
      
Correct implementation - 

      public E pop() {
         E top = elements[size--];
         elements[size] = null;
         return top;
      }
      
 We need to always define the objects to the *narrowest possible scope*.
 
 * **While implementing our own cache** - Implementing cache and not cleaning it up, when some of the objects within the cache becomes obsolete, leads to greater memory footprint and inceased performace, inturn due to greater GC activity. The best way to manage a cache is to use `WeakHashMap`. `WeakHashMap` keeps the keys which are **strongly referenced** by the running programs, deleting those which have no more references (in turn, making the keys eligible for GC).
 
 * **Observer pattern - registering listener** - Provide explicit *deregister* methods to deregister the listeners. Again make use of `WeakHashMap` to store them.
 
<a id='create_destroy_finalizers' />
 
### Avoid finalizers
 
* **Finalizers are unpredictable, often dangerous and generally unnecessary** 
* There is no guarantee that the finalizers will be executed promptly, hence it is not a good idea to **execute anythging that is time-critical** in a fianlizer.
* **Finalizer delays the reclamation of object** - If a class has implemented `finalize()` method, then the GC will wait for *finalizer thread* to finish the finalization process, before it reclaims the memory of the object. Since the *finalizer threads* are generally of low-priority, they can take arbitrarily long time to finish the finalizers.
* **There is severe performance penalty for using finalizers** - As explained by Goetz under the article - [Finalizers not your friend](https://www.ibm.com/developerworks/library/j-jtp01274/), creating and destroying an object with finalizers are ~400 times slower, due to the registration process and finalization process GC goes through, while creating and destroying the objects.
* To clean up the resources, it is good idea to provide **explicit cleanup methods**.
 
 
#### The only advantage
 
The only advantage of finalizers could be to use them as a *safety net*, incase owner of an object, forgets to call the clean up method, which will release the resources held by the object. In this case, the finalizer can *Log a warning* saying that the resources were left open! and also try to *close the resources* to avoid any resource leak (Its better late than never)
