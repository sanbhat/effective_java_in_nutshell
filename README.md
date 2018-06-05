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
   2. [Override hashCode inline with equals](#common_methods_hashcode)
   3. [Overriding toString](#common_methods_tostring)
   4. [Overriding clone judiciously](#common_methods_clone)
   5. [Consider implementing Comparable](#common_methods_compareTo)
3. [Classes and Interfaces](#ci)
   1. [Minimize the accessibility of classes and members](#ci_minimize_access)

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


<a id='common_methods_hashcode' />

### Always override hashCode() if you override equals()

The default implementation of `hashCode()` returns the memory address of the object converted to integer.

**You must override hashCode() inline with with equals(), if you have already overriden equals() method**. Following is the contract between the 2 methods as per the Java specifications

1. The value returned by the `hashCode()` method should remainin consistent throughout the exeucution of an application, provided the field used within the equals / hashCode, have not changed. However, the value can change, when the method is invoked within another execution of the application.
2. If two objects are equal (as per the `equals()` method), then the value returned by the `hashCode()` method of those two objects must be same
3. If two objects are not equal, then it is not mandatory for the hashCode method on those two objects to produce different results. In other words, objects having same hashCode value, can be unequal.

Not obeying above rules will lead to **inconsistent and unpredictable behavior**, when the object is used in hashing based datastructures such as `HashMap`, `HashSet` or `Hashtable`. 

Consider a simple class, overriding `equals()` method and not overriding `hashCode()`

      class Account {
            
            private final Integer id;
         
            public Account(Integer id) {
               this.id = id;
            }
            
            public boolean equals(Object o) {
               if(this == o) return true;
               if(!(o instanceof Account)) return false;
               Account a = (Account)o;
               return a.id != null ? a.id.equals(id) : false;               
            }
            
            //No hashCode() - BROKEN!
      }
      
Now, lets use the object as a key in `HashMap` - 

      HashMap<Account, String> map = new HashMap<>();
      map.put(new Account(1), "Test Account");
      
When you invoke `map.get(new Account(1));`, the result will be `null` because, even though the 2 objects used (in `put` and `get`) are logically equal, they have different hashCode values. Within the `HashMap`, either the new object will lookup a different hash bucket, than what it used to store the first object.  **Even if the second object hashes to same bucket, still the result will be `null` because, `HashMap` has an optimization, which will not proceed further, if the `hashCode` value is different!**.

#### The General rule

1. Start with an integer `result`, with a value `17` (randomly selected)
2. Do the following for all the significant fields `f` , which are used in the `equals` method
   1. Compute the hash of the field - `c` - using following methods
      1. if `f` is `boolean`, then `c = f ? 1 : 0`
      2. if `f` is `char`, `byte`, `short` or `int`, then `c = (int) f`
      3. if `f` is `long`, then `c = (int) (f ^ (f >>> 32))` OR, since Java8 use `Long.hashCode(f)`
      4. if `f` is `float`, then `c = Float.floatToIntBits(f)` OR, since Java8 use `Float.hashCode(f)`
      5. if `f` is `double`, then `c = Double.doubleToLongBits(f)); c = (int)(c ^ (c >>> 32));` OR, since Java8 use `Double.hashCode(f)`
      6. if `f` is an object, then similar to how you check the equality, compute the hashCode value recursively and combine it to the overall result.
      7. if `if` is an array, then use variants of `Arrays.hashCode(f)` method and again combine the result with the overall result;
    2. Once the hash value is calculated, then combine the computed value with overall `result`
         
            result = 31 * result + c;
3. return `result`
4. Check carefully, by means of unit testing to see if `hashCode()` and `equals()` behave consistently
    
    
* Also, its good to *cache* the hash value for **immutable** objects, if computing it is costly.
* We should not exclude significant objects from the computation of hashcode value, just in the name of performance! This might cause significant performance later, when we use the objects in Hash based data structures.
* Avoid documenting the *expected value* from the `hashCode()` method, which might lead clients, to write specific logic depending on the *expected* result of the `hashCode` method. This might prevent further improvements to the hashing methodology later.

<a id='common_methods_tostring' />

### Override toString()

The default implementation of `toString()` in `Object` class gives a value with the format `<class-name>@<hexadecimal-representation-of-hashcode>`

* Providing a good `toString()` implementation makes our class pleasent to use
* `toString()` method if practical, should return all the interesting information contained in the object
* The documentation of `toString()` method should clearly specify, if the method returns the values in a particular **format** or not. If the method maintains a format, then changing it later might be difficult as many clients may already been consuming the values in that format.
* One should always provide programmatic access to all the information contained in the value returned by `toString()` method of the object - This will make the code robust, as client will not try to get those information by parsing `toString()` value which can have unreliable format.


<a id='common_methods_clone' />

### Overriding clone judiciously

* `Cloneable` interface was designed to be a **mixin interface**, to advertise that the object which implements it will support cloning, but ended up with a flaw by not having the `clone()` method.
* The job of `Cloneable` interface is to allow classes implementing them, to get *field-by-field* copy when invoking Object's `clone()` method, and to throw `CloneNotSupportedException` if someone tries to invoke the `clone()` method without implementing the interface. This contract looks **extra-linguistic** and **flawed**.

#### If you anyway would like to override clone()?

* If you override the `clone` method in a non-final class, then you should return an object obtained by invoking `super.clone`. If all the super classes obey this rule, then eventually `Object`'s `clone()` method will be called, which results in creating the right object. This mechanism is isn't properly enforced via the contract!
* A class implementing `Cloneable` interface, is expected to provide a fully functional `public clone` method.
* While implementing `clone`, change the return type of the method to the concrete class type of the enclosing class. This is possible since Java 1.5's *covariant return types* and this mechanism will also uphold a general principal - **never make the client do anything that the library can do for the clients**. Also omit throwing `CloneNotSupportedException` - checked exception!
* Handle the cloning of mutable objects within the enclosing class. Provide **deep-cloning** mechanism, as per the necessity. Do not deep-clone in a recursive way. Take the help of constructors to create mutable fields of cloned object in a virgin state and then use internal helper methods to fill the same data as original object. - Example - `HashMap.clone()`
* clone architecture is incompatible with the nornal use of `final` fields referring to mutable objects. Remove `final` keywords as necessary. 

#### Better off avoiding clone() ? 

* Its better to simply avoid providing the clone capability or providing an alternative means of object copying.
* A fine approach is to provide *copy constructors* or *copy factory* - which solves many shortcomings of clone method discussed above.
* The *copy* constructors or factory methods can accept the general interfaces, which can allow copying of objects of one type to another. They are called *conversion constructors* or *conversion factories*. - Example - converting `ArrayList` to `LinkedList`, using former's constructor - `ArrayList(Collection c)`.


#### Advantage

`clone()` seems to give the best performance while copying arrays.

<a id='common_methods_compareTo' />

### Consider implementing Comparable

* The `compareTo` method is not present in `Object`class, rather it is part of `Comparable` **functional interface**.
* The method is similar to `equals` except, it provides "order comparison" in addition, and it is *generic*. By implementing `Comparable`, a class can define its *natural ordering*.
* Multiple utility methods in `Collections` and `Arrays` use the `Comparable` contract to sort and search the array of objects.

#### The general contract

The `compareTo` method takes `(T o)` - where `T` is type of class and `o` - is other object, with which `this` object is being compared. The method will return negative, zero or positive integer as this object is less than, equal to or greater than the other object. This method will throw `ClassCastException` if the other object's type, prevents it from being compared to this object.

if we consider `sign(expression)` function to return -1, 0 or 1, if the `expression` evaluates to a negative, zero or positive value respectively, then following rules needs to be kept in mind while implementing the `compareTo` method.

1. Ensure `sign(x.compareTo(y)) == -sign(y.compareTo(x))` for all `x` and `y`. Meaning, if `x` is greater than `y` then `y` should be less than `x`; if `x` is equal to `y` then `y` should be equal to `x`; if `x` is less than `y` then `y` must be greater than `x`.

2. If `x.compareTo(y)` throws `ClassCastException`, then `y.compareTo(x)` must throw the same exception.

3. Transivite relation should be maintained. If `x.compareTo(y) > 0` and `y.compareTo(z) > 0` then `x.compareTo(z)` must be `> 0` for all `x`, `y` and `z`. Meaning, if `x` is greater than `y` and `y` is greater than `z`, then it should imply that `x` must be greater than `z`.

4. It is **stongly recommended, but not strictly required** that if `x.compareTo(y) == 0` then `x.equals(y)` should be `true`.
If the equality test result of `compareTo` matches that of `equals` method, then `compareTo` method is said to be *consistent with equals* otherwise, it is said to be *incosistent with equals*.  In case of *inconsistency* a Note should be provided in the documentation stating the same.

* Since the `compareTo` contract resembles the `equals` contract discussed in the previous section, the same caveats and work-arounds apply here too.
* The difference between writing `compareTo` and `equals` is that, `compareTo` is parameterized, hence there is no need to cast the other object. We cannot compare objects of two different types. If we compare an object with null, the method is expected to throw `NullPointerException`.
* **NOTE** The general contracts defined for collection interfaces (`Collection`, `Map`, `Set`) are based on `equals` method. But the *sorted collections* use `compareTo` method to check the equality, and inconsistency between the implementations might lead to unexpected behavior. Example - Let's say, we have a class which implements `equals` and `compareTo` in different ways, and there are 2 instances of that class which are **equal** as per `equals` and **not equal** as per `compareTo`. When we add these instances to `TreeSet`, the set will retain both the instances even if they are **equal** (as per `equals`) which is against the `Set` contract.
* Compare primitive fields using `<` or `>` operator. For floating points use, `Float.compare` or `Double.compare`. For arrays, apply the guidelines for each of the elements.
* If the class has multiple significant fields, then the order in which they must be compared becomes critical. The comarison logic must start with, comparing most significant field and must move down towards least significant fields. If the comparison result at any particular stage is not zero (not equals), then the logic will halt and result should be returned immediately. 
* **Caution** - take care of **integer overflow!** Consider the following `compareTo` implementation, which, at first look, seems to be perfectly fine.

      class Task implements Comparable<Task> {
      
            private int randomId;
            
            //rest of the code
            
           /*
            * Sorts the tasks by the natural ordering of integer - randomId.
            */
            public int compareTo(Task o) {
                return this.randomId - o.randomId;
            }
      
       }

Let's say we are comparing 2 tasks, task A(randomId == Integer.MAX_VALUE) and task B (randomId == Integer.MIN_VALUE) - the expeactation from above method would be that `compareTo` method will return a positive integer, if we execute `A.compareTo(B)` ( A > B ) . But the result of the above implementation is a negative value, indicating A < B, due to integer overflow. Appropriate care must be taken while comparing large integers, which are signed.

<a id='ci' />

## Classes and Interfaces

<a id='ci_minimize_access' />

### Minimize the accessibility of classes and members

* A well designed module, always hides the internal implementation details, cleanly separating it from the API, which the clients will access. The concept of hiding internal details which are not relevant to outside clients is called *encapsulation*.
* Information hiding is important for many reasons, as it enables creation of - Decoupled systems, Easy to understand and reusable modules, which are inturn easy to develop, test, optimize and use.
* Java facilitates the *information hiding* by means of *access control* mechanism. The accessibility of an entity depends on the location of its decleration and by the use of *access modifiers* (`private`, `protected` and `public`)  on the declarations.
* The rule of thumb is to **make each class or member as inaccessible as possible** 
* For top level (non-nested) classes and interfaces, there are only two possible access levels, *package-private* and *public*. If the class is part of implementation and not public API, it should be *package private* and should not have any access modifier, infront of its declaration. In this way, the class can be modified, removed, replaced without having any subsequent impact on the clients. On the other hand, classes which are *public*, are part of API and there is an obligation to keep it forever, for the sake of compatibility.
* If the *package-private* is meant to be used only by a single class, then consider making it, `private` nested inner class of the class which is using it.
* For members, there are 4 possible access modifiers

Access modifiers | Description 
--- | --- 
**private** | The member is accessible only from top level class where it is declared
**package-private (default)** | The member is accessible from any class in the package where it is declared. The access level you get if no access modifier is specified.
**protected** | The member is accessible from the subclasses of the class where it is declared and from any class in the package where it is declared.
**public** | The member is accessible anywhere.

* After carefully designing class's public API, you should try to make all the other members private. If some members are required by classes in the same package, then `private` modifier should be removed from the declaration of that member, making it *package-private*. If you find doing this often, then redesign the API structure.
* To facilitate testing, it is acceptable to change the accessiblity of `private` members to `package-private`, but not beyond that.
* **Instance fields should never be public** - if an instance field is non-final or final pointing to a mutable object, then by making it public, you loose the ability to limit its value and fail to take an action, if its value gets modified.
* **Classes with public mutable fields** are not *Thread-safe*.
* You can have a `public static final` field, if it is an integral part of the abstraction provided by the class. Such field should refer to primitive types or immutable objects.
* If a `public static final` field refers to mutable objects (such as array with non-zero length), then it is wrong to export it as `public` or assign it a `public` access method. Care must be taken, if such access is given outside the class, for example, the access to such fields can be given using following ways -

      //Providing access to private array
      private static final Task[] TASK_ARR = { ... };
      public static final List<Task> TASK_LIST = Collections.unmodifiableList(Arrays.asList(TASK_ARR));
      
or

      //Providing access to private array
      private static final Task[] TASK_ARR = { ... };
      
      public static Thing[] getTaskArr() {
            return TASK_ARR.clone();
      }
      
