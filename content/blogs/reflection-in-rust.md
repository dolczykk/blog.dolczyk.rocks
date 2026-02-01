+++
title = "How Rust does Reflection?"
date = 2026-02-02
description = "Understanding reflection in Rust"
+++

Reflection is a key mechanism in languages with extensive runtime environments, such as C# or Java. It allows for inspection and modification of a program during its execution. However, in languages like Rust, which prioritize static analysis and performance, the situation looks different. Is reflection even possible in Rust?

## What is reflection

Reflection enables modification of a structure, function, or object fields during program execution. Thanks to this mechanism, you can access elements that are normally hidden. For example, if field `A` is private, standard code cannot access it, but using reflection you can force reading or changing its value.

Types of reflection are divided into:

- **Static** - occurs directly during compilation, e.g. macros (Rust), source generators (C#)
- **Dynamic** - executes during program runtime, e.g. reading fields from structures, rebuilding a structure by adding fields

### Comparison of reflection types

| **Dynamic reflection** | **Static reflection** |
| --- | --- |
| At program runtime | Before execution (during compilation/analysis) |
| Performance overhead at runtime due to long operation execution time | Cost during compilation (longer time, but no additional runtime overhead) |
| High dynamism, "magical" API, harder to debug | More explicit, statically generated code |

## Where is this mechanism used

The reflection mechanism is used in:

- **Serialization/Deserialization** - converting objects to/from formats such as JSON, XML, etc.
- **Data mapping** - transferring values between different models (e.g., DTOs and database entities)
- **Dependency Injection containers** - automatic dependency resolution
- **Plugins** - loading dynamic libraries (.dll, .so, .dylib) and running code contained within them
- **Testing** - detecting unit tests using attributes
- **Metaprogramming** - creating code that manipulates other code

## How does the situation look with Rust

Rust does not have built-in dynamic reflection known from C# or Java. This language relies on a metaprogramming system in the form of macros that generate code during compilation process. Although full reflection is not currently part of the language, research work is ongoing (they're experimenting on reflection and comptime feature like mostly 3rd party libraries depends that you can [read here](https://rust-lang.github.io/rust-project-goals/2025h2/reflection-and-comptime.html)).

Currently, there's a 3rd party approach for "pseudo-reflection" in Rust. There are libraries that "emulate" the behavior of dynamic reflection. The most popular ones include:

- [bevy_reflect](https://crates.io/crates/bevy_reflect)
- [Facet](https://facet.rs/)

## Example of operation

In this article, I will demonstrate an example of "pseudo-reflection" using the [bevy_reflect](https://crates.io/crates/bevy_reflect) library, because it is a very simple library to use and offers considerable possibilities (yea this library is from game engine called [Bevy](https://bevy.org/)).

To perform example reflection, you need to inject the Reflection trait into the derive macro for the selected structure.

Let's define a player structure:

```rust
#[derive(Reflect)]
struct Player {
    name: String,
    hp: u32,
    score: u32
}
```

Above we defined fields such as:

- `name` - player's name
- `hp` - player's health
- `score` - the score the player has currently achieved

Let's add an implementation of the `new` constructor, setting default values:

```rust
impl Player {
    fn new(name: String) -> Player {
        Player {
            name,
            hp: 100,
            score: 0
        }
    }
}
```

Having such a defined structure, we can perform a field overwrite operation (even if it were private in another module):

```rust
fn main() {    
    let mut player = Player::new("Pikachu".to_string());

    let mut hp = player.get_hp();
    println!("Current hp: {}", hp); // Current hp: 100

    player.field_mut("hp").unwrap().apply(&42u32);

    hp = player.get_hp();
    println!("Updated hp: {}", hp); // Current hp: 42
}
```

The most detailed line in this solution:

```rust
player.field_mut("hp").unwrap().apply(&42u32);
```

Here we call the `field_mut` method, which with the given parameter `"score"` will return a mutable reference to the `score` field, and then we can overwrite it with any value we want.

## How is it possible that reflection works this way

Unlike languages with a runtime like C# or Java, which allow memory inspection during program execution, `bevy_reflect` relies on code generation at compile time.

When we add `#[derive(Reflect)]`, the macro creates a trait implementation inside the same struct. Thanks to this, the generated code has legal access to private fields and exposes them externally through a safe, public interface. This is not magic breaking language rules, but a clever use of the macro system to automate access.

## Operations on dynamic types

`bevy_reflect` allows us to create dynamic types during program execution. Types such as the following have been implemented:

- DynamicTuple
- DynamicArray
- DynamicList
- DynamicMap
- DynamicStruct
- DynamicTupleStruct
- DynamicEnum

Let's recreate the `Player` structure as a `DynamicStruct`:

```rust
fn main() {
    let mut player = DynamicStruct::default();

    player.insert("name", "Geralt".to_string());
    player.insert("score", 35u32);
    player.insert("hp", 100u32);

    // Struct fields: DynamicStruct(_ { name: "Geralt", score: 35, hp: 100 })
    println!("Struct fields: {:?}", player);
}
```

Now let's add more lines that will overwrite the score field:

```rust
fn main() {
    // ...Continuation of upper implementation

    // Output: Actual score field: 35
    println!("Actual score field: {:?}", player.get_field::<u32>("score").unwrap());

    player.field_mut("score").unwrap().apply(&200u32);

    // Output: Updated score field: 200
    println!("Updated score field: {:?}", player.get_field::<u32>("score").unwrap());
}
```

The downside is that we cannot write our own function implementation (although it is possible using `DynamicFunction`) for this struct, which is related to the limitations of such a library.

## Let's create something more advanced

For the implementation of this mini project, I chose the goal of creating a simple DI container that will inject dependencies into fields using reflection. It's known that DI implementation is rarely seen in Rust, due to the strong type system, modularity and ownership, but this is a great example of how reflection is used in practice.

Let's create the `DiContainer` structure:

```rust
#[derive(Debug)]
pub struct DiContainer {
    dependencies: HashMap<String, Arc<dyn Reflect>>,
}
```

Above we declare a hash map of dependencies, which will hold the path to the type itself as a key and the value will be any object that implements the Reflect trait with the possibility of sharing across multiple threads.

The new method for the structure looks as follows:

```rust
impl DiContainer {
    pub fn new() -> Self {
        DiContainer {
            dependencies: HashMap::new(),
        }
    }
}
```

Nothing interesting here, but now let's try to implement the register function:

```rust
impl DiContainer {
    pub fn register<T>(&mut self, service: T)
    where
        T: Reflect + Clone + FromReflect + Typed + GetTypeRegistration + TypePath,
    {
        // Make option type available for injection
        let some_service = Some(service);

        // Register the Option<T> type to easily inject optional dependencies
        let opt_type_path = format!("core::option::Option<{}>", T::type_path());

        // Add dependency to the container
        self.dependencies
            .insert(opt_type_path, Arc::new(some_service));
    }
}
```

It's getting more interesting now - isn't it? :)

The function accepts any generic that implements traits related to the bevy_reflect library. As you can see with the `opt_type_path` variable, `core::option::Option<{}>` is used, and that's because it will allow us for easier error handling when somehow it doesn't find a given struct in the container.

Let's move on to the last culminating point which is the `inject_dependencies` method:

```rust
impl DiContainer {
    pub fn inject_dependencies<T: Reflect>(&self, instance: &mut T) {
        // Get a mutable reference to the reflectable instance
        let target = instance.as_reflect_mut();

        // Ensure the target is a struct to access its fields
        if let ReflectMut::Struct(s) = target.reflect_mut() {
            for i in 0..s.field_len() {
                // Get the field and its type path
                let field = s.field_at(i).unwrap();
                let field_type_path = field.reflect_type_path().to_string();

                // If a matching service is found, inject it into the field
                if let Some(service) = self.dependencies.get(&field_type_path) {
                    s.field_at_mut(i).unwrap().apply(service.as_ref());
                }
            }
        }
    }
}
```

As the method name suggests, this is where operations related to injecting dependencies into appropriate types will happen. First, we need to extract `dyn Reflect` from the instance, then check if our `target` variable is of type Struct. If so, we iterate over the fields and simultaneously retrieve the path to the type. If we manage to find a dependency in the container, then the value is injected into the field and that's it. The whole philosophy explained, but we need to move on to some specifics, so let's create an example logger like this:

```rust
#[derive(Reflect, Default, Clone)]
pub struct Logger {
    app_name: String,
}

impl Logger {
    // Implement constructor for Logger
    pub fn new(app_name: &str) -> Self {
        Logger {
            app_name: app_name.to_string(),
        }
    }

    // Function which logs messages with the application name as a prefix
    pub fn log(&self, message: &str) {
        println!("[{}]: {}", self.app_name, message);
    }
}
```

The Logger structure has the `app_name` field and only the `new` and `log` functions, where the latter displays messages to the console in the following format:

```rust
[app_name]: Test message
```

We already have the Logger behind us, so let's create two example entities User and Product:

```rust
#[derive(Debug)]
pub struct User {
    pub id: u32,
    pub name: String,
}

impl User {
    // Implement constructor for User
    pub fn new(id: u32, name: &str) -> Self {
        User {
            id,
            name: name.to_string(),
        }
    }
}

impl Display for User {
    // Implement Display trait for pretty-printing User data
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "User ID: {}, Name: {}", self.id, self.name)
    }
}
```

```rust
#[derive(Debug)]
pub struct Product {
    id: u32,
    name: String,
    price: f64,
}

impl Product {
    // Implement constructor for Product
    pub fn new(id: u32, name: &str, price: f64) -> Self {
        Product {
            id,
            name: name.to_string(),
            price,
        }
    }
}

impl Display for Product {
    // Implement Display trait for pretty-printing Product data
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "Product ID: {}, Name: {}, Price: ${:.2}",
            self.id,
            self.name,
            self.price
        )
    }
}
```

In `User` we store id and name, while in `Product` we have `id`, `name`, and `price` (for now we assume the `f64` type for simplicity).

Now let's implement the services:

```rust
#[derive(Reflect, Default)]
pub struct UserService {
    logger: Option<Logger>,

    #[reflect(ignore)]
    users: Vec<User>,
}

impl UserService {
    pub fn add_user(&mut self, user: User) {
        // Check if logger is available and log the action
        if let Some(logger) = self.logger.as_ref() {
            logger.log("Adding a new user");
        }

        self.users.push(user);
    }

    pub fn list_users(&self) {
        // Check if logger is available and log the action
        if let Some(logger) = self.logger.as_ref() {
            logger.log("Listing all users");
        }

        // Print each user
        for user in &self.users {
            println!("{}", user);
        }
    }
}

```

```rust
#[derive(Reflect, Default)]
pub struct ProductService {
    logger: Option<Logger>,

    #[reflect(ignore)]
    products: Vec<Product>,
}

impl ProductService {
    pub fn add_product(&mut self, product: Product) {
        // Check if logger is available and log the action
        if let Some(logger) = self.logger.as_ref() {
            logger.log("Adding a new product");
        }

        // Add product to the products list
        self.products.push(product);
    }

    pub fn list_products(&self) {
        // Check if logger is available and log the action
        if let Some(logger) = self.logger.as_ref() {
            logger.log("Listing all products");
        }

        // Print each product
        for product in &self.products {
            println!("{}", product);
        }
    }
}

```

You have certainly noticed the use of `#[reflect(ignore)]`, which in this case lets the mechanism know to ignore these fields during iteration over multiple fields in the structure in ordinary circumstances.

We have the two services above, which allow us to add and display products or users. Now we just need to wrap everything below:

```rust
fn main() {
    // Create the DI container
    let mut container = DiContainer::new();

    // Create service instances
    let mut products_service = ProductService::default();
    let mut users_service = UserService::default();

    // Register services in the container
    let logger = Logger::new("kiroshi");
    container.register(logger);

    // Inject dependencies into instances
    container.inject_dependencies(&mut products_service);
    container.inject_dependencies(&mut users_service);

    // Add products
    let product1 = Product::new(1, "Opti-Flash MK.II", 600.99);
    let product2 = Product::new(2, "Cockatrice Optics", 799.09);

    products_service.add_product(product1);
    products_service.add_product(product2);

    // Add users
    let user1 = User::new(1, "Vincent");
    let user2 = User::new(2, "Panam");

    users_service.add_user(user1);
    users_service.add_user(user2);

    // List all products and users
    products_service.list_products();
    users_service.list_users();
}
```

The console output looks as follows:

```text
[kiroshi]: Adding a new product
[kiroshi]: Adding a new product
[kiroshi]: Adding a new user
[kiroshi]: Adding a new user
[kiroshi]: Listing all products
Product ID: 1, Name: Opti-Flash MK.II, Price: $600.99
Product ID: 2, Name: Cockatrice Optics, Price: $799.09
[kiroshi]: Listing all users
User ID: 1, Name: Vincent
User ID: 2, Name: Panam
```

As you can see, we managed to create a DI container in Rust using reflection. Below I leave a link to the [repository](https://github.com/dolczykk/di-bevy-container) where you can browse the structure of the entire project.

## Library limitations

Despite considerable possibilities, this solution has its drawbacks:

- **Accepting only types with `'static` lifetime** - there is no way to accept types with a shorter lifetime, e.g., `&'a T`
- **Dynamic types cannot be cast** - e.g., `DynamicStruct` to a regular struct isn't achievable
- **No automatic registration of `dyn Reflect`** - you have to add it manually to make reflection possible

## When NOT to use reflection

- **When performance matters** - algorithms based on reflection are slower than static code
- **Type safety** - reflection by nature bypasses some compiler guarantees. Errors may only appear at runtime
- **If you want clear code** - excessive use of "magic" makes it difficult to understand data flow. IDE may have problems tracking the usage of fields modified only through reflection

## Summary

Reflection in Rust is possible, although it works on a different principle than in .NET or Java. Instead of introspection supported by a VM, we are dealing with a powerful macro system generating supporting code. Libraries like `bevy_reflect` allow for creating advanced architectures, however, they should be used consciously, remembering about performance costs and the loss of some static safety guarantees.
