# Context-Generic Programming (CGP) Overview

Context-Generic Programming (CGP) is a modular programming paradigm that enables zero-cost abstractions through compile-time dispatch and type-level programming. It provides a powerful framework for building extensible, composable software architectures in Rust.

## Core Philosophy

CGP addresses the challenge of building modular systems that remain performant by moving all dispatch logic to compile time. Instead of runtime polymorphism through trait objects or function pointers, CGP uses Rust's type system to wire components together during compilation, resulting in zero-cost abstractions.

## Fundamental Concepts

### 1. Contexts

Contexts are the central units in CGP that encapsulate state and behavior. They serve as the primary data structures that components operate on.

```rust
#[cgp_context(MyContextComponents)]
#[derive(HasField)]
pub struct MyContext {
    pub name: String,
    pub counter: u64,
}
```

Key characteristics:
- Hold application state and data
- Can inherit component implementations through presets
- Serve as the target for component operations
- Enable type-safe field access

### 2. Components

Components are named capabilities that contexts can have, defined as traits. They represent abstract operations or behaviors.

```rust
#[cgp_component(Greeter)]
pub trait CanGreet {
    fn greet(&self);
}

#[cgp_getter(NameGetter)]
pub trait HasName {
    fn name(&self) -> &str;
}

#[cgp_type]
pub trait HasErrorType {
    type Error: Debug;
}
```

Components come in three main varieties:
- **Method components**: Define behavior through methods
- **Getter components**: Provide access to data
- **Type components**: Define associated types

### 3. Providers

Providers contain the concrete implementations of components. They are wired to contexts through the delegation system.

```rust
#[cgp_provider]
impl<Context> Greeter<Context> for GreetHello
where
    Context: HasName,
{
    fn greet(context: &Context) {
        println!("Hello, {}!", context.name());
    }
}
```

Providers can be:
- **Custom implementations**: Hand-written logic
- **Built-in providers**: Like `UseField`, `UseType`, `UseContext`
- **Delegating providers**: Like `UseDelegate` for dynamic dispatch

## Architecture Deep Dive

### Component System

The component system is built on several core traits that work together:

1. **`HasCgpProvider`**: Links contexts to their provider configuration
2. **`DelegateComponent<Name>`**: Creates type-level mappings from component names to providers
3. **`IsProviderFor<Component, Context, Params>`**: Ensures proper constraint propagation
4. **`CanUseComponent<Component, Params>`**: Verifies component availability at compile time

### Delegation Patterns

**UseDelegate Pattern**: Enables dispatching based on generic type parameters:

```rust
#[cgp_component {
    provider: ErrorRaiser,
    derive_delegate: UseDelegate<SourceError>,
}]
pub trait CanRaiseError<SourceError>: HasErrorType {
    fn raise_error(error: SourceError) -> Self::Error;
}
```

**UseContext Pattern**: Forwards provider implementation back to consumer traits:

```rust
impl<Context> Greeter<Context> for UseContext
where
    Context: CanGreet,
{
    fn greet(context: &Context) {
        context.greet()
    }
}
```

### Component Wiring

Components are wired to contexts using declarative configuration:

```rust
delegate_components! {
    MyContextComponents {
        GreeterComponent: GreetHello,
        NameGetterComponent: UseField<symbol!("name")>,
        [
            FooTypeComponent,
            BarTypeComponent,
        ]: UseType<String>,
    }
}
```

This creates the necessary `DelegateComponent` implementations that map component names to their providers.

## Preset System and Inheritance

### Defining Presets

Presets are reusable component configurations that promote code reuse:

```rust
cgp_preset! {
    BasicPreset {
        ErrorTypeComponent: UseType<anyhow::Error>,
        LoggerComponent: PrintLogger,
    }
}
```

### Multiple Inheritance

CGP supports multiple inheritance with explicit conflict resolution:

```rust
cgp_preset! {
    AdvancedPreset: BasicPreset + NetworkPreset {
        override LoggerComponent: StructuredLogger,
        CacheComponent: RedisCache,
    }
}
```

### Context Integration

Contexts can inherit from presets:

```rust
#[cgp_context(MyContextComponents: AdvancedPreset)]
pub struct MyContext {
    pub database: Database,
    pub config: Config,
}
```

## Advanced Features

### Type-Level Programming

CGP extensively uses type-level constructs for compile-time configuration:

- **Type-level strings**: `symbol!("field_name")` for field identification
- **Product types**: `Product![A, B, C]` for heterogeneous lists
- **Field access**: `HasField<Tag>` for type-safe field operations

### Generic Dispatching

The delegation system supports dispatching based on type parameters:

```rust
delegate_components! {
    MyComponents {
        SerializerComponent: UseDelegate<
            new SerializerMapping {
                Json: JsonSerializer,
                Yaml: YamlSerializer,
                Toml: TomlSerializer,
            }
        >,
    }
}
```

### Error Handling

CGP provides sophisticated error handling through the type system:

```rust
#[cgp_type]
pub trait HasErrorType {
    type Error: Debug;
}

#[cgp_component {
    derive_delegate: UseDelegate<SourceError>,
}]
pub trait CanRaiseError<SourceError>: HasErrorType {
    fn raise_error(error: SourceError) -> Self::Error;
}
```

## Code Generation

CGP relies heavily on procedural macros for ergonomic APIs:

### Primary Macros

1. **`#[cgp_component]`**: Generates provider traits, implementations, and component names
2. **`#[cgp_context]`**: Creates provider configurations and links contexts to components
3. **`#[cgp_provider]`**: Adds constraint propagation to provider implementations
4. **`delegate_components!`**: Creates type-level component mappings
5. **`cgp_preset!`**: Defines inheritable component configurations

### Compile-Time Verification

CGP includes compile-time verification to catch configuration errors:

```rust
check_components! {
    CanUseMyContext for MyContext {
        GreeterComponent,
        NameGetterComponent,
        ErrorTypeComponent,
    }
}
```

## Key Benefits

### Performance
- **Zero-cost abstractions**: All dispatch happens at compile time
- **No runtime overhead**: Component resolution is statically determined
- **Optimal code generation**: Compiler can inline and optimize across component boundaries

### Modularity
- **Composable components**: Mix and match capabilities as needed
- **Preset inheritance**: Share common configurations across contexts
- **Extensible architecture**: Add new providers without modifying existing code

### Safety
- **Type safety**: Incorrect component wiring caught at compile time
- **Constraint propagation**: Clear error messages for missing dependencies
- **Memory safety**: No runtime dispatch means no trait objects or function pointers

### Developer Experience
- **Declarative configuration**: Express component relationships clearly
- **Automatic implementations**: Reduce boilerplate through code generation
- **Clear error messages**: Compile-time feedback guides correct usage

## Real-World Applications

CGP is particularly well-suited for:

- **Plugin architectures**: Where capabilities need to be mixed and matched
- **Protocol implementations**: Where different providers handle various protocol variants
- **Configuration systems**: Where different deployment environments need different implementations
- **Testing frameworks**: Where mock providers can replace real implementations
- **Microservice orchestration**: Where different services provide different capabilities

## Comparison to Other Patterns

### vs. Trait Objects
- **CGP**: Compile-time dispatch, zero cost, but larger binary size
- **Trait objects**: Runtime dispatch, small binary, but runtime overhead

### vs. Generics
- **CGP**: Structured component system with inheritance and presets
- **Generics**: Simple type parameterization without higher-level organization

### vs. Dependency Injection
- **CGP**: Compile-time wiring with type safety
- **DI containers**: Runtime wiring with potential for configuration errors

## Conclusion

Context-Generic Programming represents a sophisticated approach to modular programming in Rust. By leveraging the type system for compile-time configuration and dispatch, CGP enables the creation of highly modular, performant, and type-safe applications. While it requires learning new concepts and patterns, the benefits in terms of performance, safety, and modularity make it a powerful tool for building complex Rust applications.

The extensive use of procedural macros ensures that the complexity of the underlying type-level programming is hidden behind ergonomic APIs, making CGP accessible to developers while providing the full power of compile-time dispatch and zero-cost abstractions.