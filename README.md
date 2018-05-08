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
2. [Methods common to All Objects](#common_methods)
   1. [Overriding equals](#common_methods_equals)

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


<a id='common_methods'/>

## Methods common to All Objects

<a id='common_methods_equals' />

### Overriding equals
Overriding `equals()` in a wrong way is worse than leaving it as is. Sometimes, the default `equals()` implementation - which compares the instances / references, makes sense. Following are those scenarios, where overriding `equals()` doesn't make sense.

* Each instance of the class is inherantly unique. Example - `Thread`
* You don't care about a class's logical equality. Example - `Random`
* A super class has already provided an implementation of  `equals()` whose behavior is appropriate for sub classes. Example -  `AbstractList` and `List`
* The class is private or package-private (default), and you know that `equals()` method, will not be invoked.

We overrode `equals()` method, when a class has notion of ***logical equality*** or when it is a *value* class (such as `Integer`, `Long` etc). We also implement `equals()` if the super class has not already implemented it. Following are the rules to take care of, while overriding `equals()`.

1. **Reflexive** - for any non-null reference `x`, `x.equals(x)` must be `true`.
2. **Symmetric** - for any non-null reference `x` and `y`, if `x.equals(y)` is `true` then `y.equals(x)` must also be `true`.
3. **Transitive** - for any non-null reference `x`, `y` and `z`, if `x.equals(y)` is `true` and `y.equals(z)` is `true` then `x.equals(z)` must return `true`.
4. **Consistent** - for any non-null reference `x` and `y`, multiple invocations of `x.equals(y)` must consistently return `true` (or `false`), unless their values (which are used to determine the equality) are changed
5. **Null Equality** - For any non-null reference `x` , `x.equals(null)` must return `false`.

**Reflexive** - property can be implementing by having a check in the `equals()` method to see if the parameter and `this` object are *equal by reference*

**Symmetric** - Below code breaks symmetry!

      public final class CaseInsensitiveString {
         private final String s;
         public CaseInsensitiveString(String s) {
            if (s == null)
               throw new NullPointerException();
            this.s = s;
         }
         
         // Broken - violates symmetry!
         @Override public boolean equals(Object o) {
            if (o instanceof CaseInsensitiveString)
               return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
            if (o instanceof String) // One-way interoperability!
               return s.equalsIgnoreCase((String) o);
            return false;
         }
         
      }

because, for 2 objects `CaseInsensitiveString s1 = new CaseInsensitiveString("abc");` and `String s2 = "abc;"`, while `s1.equals(s2)` is `true`,  `s2.equals(s1)` is `false`.

**Transitive** - We need to be careful while adding *value* field to a subtype of a class (which itself represents a value). For example consider a value class called `Point`

      class Point {
            private final int x;
            private final int y;
            
            public Point(int x, int y) {
               this.x = x;
               this.y = y;
            }
            
            @Override
            public boolean equals(Object o) {
               if(!(o instanceof Point)) {
                  return false;
               }
               
               Point other = (Point) o;
               return this.x == other.x && this.y == other.y;
            }
            
      }

Let's create a new class which has *logical equality*, by adding another  *value* field called `Color`.

      class ColorPoint extends Point {
            private final Color color;
            
            public ColorPoint(int x, int y, Color color) {
               super(x, y);
               this.color = color;
            }
      }
      
Now think, how the `equals()` method should be defined?

* We cannot live with the super class implementation - this will do Color blind comparison!
* Let's try a simple implementation

         // Broken - violates symmetry!
         @Override public boolean equals(Object o) {
            if (!(o instanceof ColorPoint))
               return false;
            return super.equals(o) && ((ColorPoint) o).color == color;
         }
         
This will break, Symmetry! - Let's say we have `ColorPoint c = new ColorPoint(1, 2, Color.RED)` and `Point p = new Point(1,2)`, `p.equals(c)` will be **true**, but `c.equals(p)` will be **false**.
         
* Let's try to fix above issue

      @Override
      public boolean equals(Object o) {
            if(!(o instanceof Point)) return false;
            
            if(!(o instanceof ColorPoint)) {
               return o.equals(this); // Color blind comparison!
            }
            
            ColorPoint c = (ColorPoint) o;
            return super.equals(c) && c.color == this.color;
      }

This will maintain the Symmetry but at the cost of Transitivity! - Let's say we have 

         ColorPoint c1 = new ColorPoint(1, 2, Color.RED);
         Point p = new Point(1,2);
         ColorPoint c2 = new ColorPoint(1, 2, Color.BLUE);

Here `c1.equals(p)` is **true**, `p.equals(c2)` is **true**, but `c1.equals(c2)` is **false**.

* Forget `instaceof` , why not check class type?

         @Override public boolean equals(Object o) {
               if (o == null || o.getClass() != getClass())
                  return false;
               Point p = (Point) o;
               return p.x == x && p.y == y;
         }

This will break ***Liskov Principle*** which says, any significant property of a supertype must hold good for subtype too. So if we subtype the class `Point` by adding some fields and NOT overriding `equals()` method, then the sub-type will not be *logically equal* with the super type instance, even if it has not added any *value* field.

Hence, the best possible way of implementing `equals()` while adding *value* field in sub-type is to use **Composition**.

      class ColorPoint {
            
            private final Point p;
            private final Color c;
            
            public ColorPoint(int x, int y, Color c) {
               this.p = new Point(x, y);
               this.c = c;
            }
      
            @Override
            public boolean equals(Object o) {
               if(!(o instanceof ColorPoint)) {
                  return false;
               }
               
               ColorPoint cp = (ColorPoint) o;
               return this.p.equals(cp.p) && this.c == cp .c;
            }
      }


**Consistency** - For `equals()` method of a class to be consistent, the class should be *Immutable* or the equals method should not depend on unreliable resources.

**Null Equality** - It is not required to check if the parameter of `equals` method `== null` to return `false`. The `instanceof` check servers both the purpose (checking null and type compatibility).

#### The General rule

1. Start comparing other object with this, using reference comparison `==`.
2. Check for type compatibility (and null check) using `instanceof`.
3. Cast the argument to correct type.
4. For significant fields of other object, check the corresponding fields of this object.
5. Compare primitive types using `==` except `float`, `double`. For those, use `Float.compare(..)` and `Double.compare(..)` respectively.
6. For fields which are of type objects, invoke equals recursively.
7. Do a null check on the object fields, `(field == this.field) || (field != null && field.equals(this.field)`.
8. After the implementation, use *Test cases* to assert if the equals implementation is *Reflexive*, *Symmetric* and *Transitive*.
9. Do not overload equals with strong types - `public boolean equals(Myclass o)` - use `@Override` to detect such error.
