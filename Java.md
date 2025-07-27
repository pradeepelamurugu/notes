# Java, Spring, and Spring Boot Notes

## Java Core Concepts

* **WORA (Write Once, Run Anywhere):** Java code, once compiled into bytecode, can run on any platform that has a Java Virtual Machine (JVM).
* **JRE, JVM, and JDK:**
    * **JRE (Java Runtime Environment):** Required to run Java code. It includes the JVM and core libraries.
    * **JVM (Java Virtual Machine):** The abstract machine that executes Java bytecode.
    * **JDK (Java Development Kit):** Needed for development. It includes the JRE along with development tools like the compiler (javac).
* **`StringBuffer` vs. `StringBuilder`:**
    * `StringBuffer` is **thread-safe** (synchronized methods), making it suitable for multi-threaded environments but potentially slower.
    * `StringBuilder` is **not thread-safe** (non-synchronized methods), offering better performance in single-threaded scenarios.
* **`Object` Class:** Every class in Java implicitly extends the `Object` class, making its methods available to all objects.
* **Static Blocks:** A `static` block is executed only once, when the class is loaded into memory (not when an object is instantiated).
* **`java.lang` Package:** This package is implicitly imported into every Java class and contains fundamental classes like `System`, `String`, `Object`, etc.
* **Default Access Modifier:** If a variable or method in a class is declared without an explicit access modifier, it defaults to **package-private** (or "default" access). This means it's accessible only within the same package.
* **Dynamic Method Dispatch (Runtime Polymorphism):** Allows a superclass reference variable to refer to a subclass object. The method to be invoked is determined at runtime based on the actual object type.
    ```java
    Parent obj = new Child(); // Reference of superclass, object of subclass
    ```
* **Wrapper Classes:** Provide a way to use primitive data types as objects.
    * `int` -> `Integer`
    * `char` -> `Character`
    These are part of the `java.lang` package.
* **Autoboxing:** Automatic conversion of a primitive type to its corresponding wrapper class.
    ```java
    int num = 7;
    Integer num1 = num; // Autoboxing: int to Integer
    ```
* **Auto-unboxing:** Automatic conversion of a wrapper class object to its corresponding primitive type.
    ```java
    Integer num1 = new Integer(10);
    int num2 = num1; // Auto-unboxing: Integer to int
    ```
* **Abstract Classes:** Cannot be instantiated directly. They can have abstract methods (without implementation) and concrete methods. Subclasses must implement all abstract methods unless they are also declared abstract.
* **Interfaces:**
    * All methods inside an interface are implicitly `public abstract` (prior to Java 8 default/static methods).
    * All variables inside an interface are implicitly `public static final` and **must be initialized** at declaration.
* **Types of Interfaces:**
    * **Normal Interface:** Contains two or more abstract methods.
    * **Functional Interface (SAM - Single Abstract Method):** Contains exactly one abstract method. These can be annotated with `@FunctionalInterface` and are used extensively with lambda expressions.
    * **Marker Interface:** An interface with no methods or fields (e.g., `Serializable`, `Cloneable`). It serves to "mark" a class, indicating a special capability to the JVM or compiler.
* **Exception Hierarchy:** `Object` -> `Throwable` -> `Exception` (checked exceptions) / `Error` (unchecked, unrecoverable)
    * `Exception` has subclasses like `RuntimeException` (unchecked) and others like `IOException`, `SQLException` (checked). `ArithmeticException` is a common `RuntimeException`.
* **Ducking Exceptions (`throws` keyword):** Used to declare that a method might throw a certain type of exception, delegating the responsibility of handling it to the calling method.
* **Thread States:**
    * **New:** A thread has been created but not yet started.
    * **Runnable:** A thread that is ready to run and waiting for CPU time.
    * **Running:** A thread that is currently executing.
    * **Waiting/Blocked:** A thread that is temporarily inactive (e.g., waiting for I/O, a lock, or another thread to complete).
    * **Timed Waiting:** A thread that is waiting for a specified amount of time.
    * **Terminated/Dead:** A thread that has completed its execution.
* **Stream API:**
    * A stream can be consumed **only once**. Attempting to re-use a stream after it has been closed or consumed will result in a `IllegalStateException`.
    * Common stream operations include `forEach`, `filter`, `map`, `reduce`, `collect`, etc.
    ```java
    Stream<Integer> s1 = List.of(1, 2, 3).stream();
    s1.forEach(n -> System.out.println(n)); // Consumes the stream
    // s1.forEach(n -> System.out.println(n)); // Throws IllegalStateException if run again
    ```

## Maven

* **Project Management Tool:** Maven is primarily used for build automation and project management.
* **`pom.xml`:** The Project Object Model (POM) file (`pom.xml`) is the core configuration file in a Maven project, where dependencies, plugins, and build configurations are defined.
* **Effective POM:** Maven internally constructs an "effective POM" by merging the project's `pom.xml` with parent POMs and inherited settings. You can view the effective POM in IntelliJ by right-clicking `pom.xml` > Maven > Show Effective POM.

## Spring Framework

* **Beans:** Objects that are instantiated, assembled, and managed by the Spring IoC (Inversion of Control) container.
* **Hibernate:** A popular Object-Relational Mapping (ORM) framework that simplifies database interactions by mapping Java objects to database tables.
* **`@Scope` Annotation:**
    * `@Scope("prototype")`: A new instance of the bean is created every time it is requested from the IoC container.
    * `@Scope("singleton")`: (Default) Only one shared instance of the bean is created and managed by the IoC container.
* **JPA (Java Persistence API) Annotations:**
    * `@Entity`: Marks a Java class as an entity, mapping it to a database table.
    * `@Id`: Designates a field as the primary key of the entity.
* **`Optional` Class:** A container object that may or may not contain a non-null value. It helps in avoiding `NullPointerException`s.
    ```java
    Optional<Student> s = repo.findById(104);
    System.out.println(s.orElse(new Student())); // Prints existing student or a new Student if not found
    ```
* **Lombok Annotations:** A library that reduces boilerplate code.
    * `@Data`: Generates getters, setters, `equals()`, `hashCode()`, and `toString()` methods.
    * `@NoArgsConstructor`: Generates a constructor with no arguments.
    * `@AllArgsConstructor`: Generates a constructor with arguments for all fields.

## Spring Boot

* **`@CrossOrigin` Annotation:** Used on controller classes or methods to handle Cross-Origin Resource Sharing (CORS) issues, allowing requests from different origins.
* **`@RequestPart` Annotation:** Used to bind parts of a multipart request to method parameters, commonly for file uploads.
* **Spring Data REST:** A Spring project that automatically generates RESTful endpoints for your JPA repositories, eliminating the need to write separate service and controller layers for basic CRUD operations.
* **Aspect-Oriented Programming (AOP):** Complements Object-Oriented Programming (OOP) by allowing separation of cross-cutting concerns (e.g., logging, security, transaction management) from the main business logic.
    * **Join Point:** A point in the execution of the program where an aspect can be applied (e.g., method execution, exception handling). "When" to apply the advice.
    * **Advice:** The action taken by an aspect at a particular join point. "What" to do.
    * **Aspect:** A modularization of a cross-cutting concern, often implemented as a class. "Where" to apply.
* **Dispatcher Servlet (Front Controller):** In Spring MVC, the `DispatcherServlet` acts as the front controller. All incoming requests first go through it, and it then dispatches them to the appropriate controller method.
* **Spring Security Integration:** When Spring Security is added, a chain of security filters typically processes requests *before* they reach the `DispatcherServlet`, handling authentication and authorization.
* **Principal:** Represents the currently authenticated user in a security context.

## Security Concepts

* **Symmetric-Key Cryptography:** Uses a single, shared secret key for both encryption and decryption (e.g., AES).
* **Asymmetric-Key Cryptography (Public-Key Cryptography):** Uses a pair of mathematically related keys: a public key for encryption and a private key for decryption (e.g., RSA).
* **Digital Signature:** Used to verify the authenticity and integrity of a message.
    1.  The sender (A) hashes the message.
    2.  A encrypts the hash with their private key (this is the digital signature).
    3.  A sends the message and the digital signature to the receiver (B).
    4.  B decrypts the digital signature using A's public key to retrieve the hash.
    5.  B also hashes the received message.
    6.  If the two hashes match, B can be confident that the message was sent by A and has not been tampered with. (Note: The user's description of encrypting with B's public key in the original note is not standard for digital signatures, which primarily focus on sender authentication).

## Deployment

* **AWS Elastic Beanstalk:**
    * For deploying Java applications with a `.jar` file, select the "Java" platform.
    * For deploying web applications packaged as a `.war` file, select the "Tomcat" platform.
