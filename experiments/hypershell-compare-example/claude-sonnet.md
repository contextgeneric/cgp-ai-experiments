# Understanding Hypershell's Compare and Branch Example: Control Flow Through Types

If you've ever wondered how to implement conditional logic in a type-level DSL, Hypershell's `compare_and_branch.rs` example offers a brilliant demonstration. This example showcases one of the most sophisticated features of Hypershell: the ability to create custom control flow constructs that operate entirely at the type level while executing real-world tasks asynchronously.

## The Big Picture: What This Example Does

At its core, this example downloads content from two different URLs, computes their SHA256 checksums, compares those checksums for equality, and then prints different messages based on whether they match. What makes this remarkable is that the entire control flow—the comparison and branching logic—is defined using Rust's type system and executed through Hypershell's component system.

The example compares two URLs: `"https://nixos.org/manual/nixpkgs/unstable/"` and `"https://nixos.org/manual/nixpkgs/unstable"` (note the trailing slash difference). In practice, these URLs likely redirect to the same content, so their checksums should be identical, causing the program to print "the checksums are equals".

## Dissecting the Type-Level DSL

The heart of this example lies in its type definitions, which read almost like pseudocode but compile to efficient, zero-cost abstractions. Let's start with the reusable `GetChecksumOf` type:

```rust
pub type GetChecksumOf<Url> = hypershell! {
    StreamingHttpRequest<
        GetMethod,
        Url,
        WithHeaders[ ],
    >
    | Checksum<Sha256>
    | BytesToHex
};
```

This type alias defines a reusable pipeline that takes a URL as a type parameter and creates a three-stage process. First, it makes a streaming HTTP GET request to the provided URL. The response stream then flows through a SHA256 checksum calculator, which processes the bytes incrementally without loading the entire response into memory. Finally, the raw bytes of the checksum are converted to a hexadecimal string representation.

The beauty of this abstraction is that `GetChecksumOf` can be instantiated with any URL type—whether it's a static string, a field reference, or any other argument type that Hypershell understands. This demonstrates Hypershell's composability: complex operations can be packaged into reusable components that work seamlessly with the rest of the type-level DSL.

## The Main Program: Conditional Execution

The main program type showcases Hypershell's custom control flow constructs:

```rust
pub type Program = hypershell! {
    If<
        Compare<
            GetChecksumOf<FieldArg<"url_a">>,
            GetChecksumOf<FieldArg<"url_b">>,
        >,
        SimpleExec<
            StaticArg<"echo">,
            WithArgs[
                StaticArg<"the checksums are equals">,
            ],
        >,
        SimpleExec<
            StaticArg<"echo">,
            WithArgs[
                StaticArg<"the checksums are not equal">,
            ],
        >,
    >
    | StreamToStdout
};
```

This type definition encodes a complete conditional execution flow. The `If` construct takes three type parameters: a condition, a "then" branch, and an "else" branch. The condition is a `Compare` operation that takes two sub-programs as arguments—in this case, two instances of `GetChecksumOf` that will fetch and compute checksums for different URLs.

What's particularly elegant is how the type system enforces the structure of the conditional. The Rust compiler guarantees that all three branches of the `If` statement are present and well-typed before any code execution begins. This means that many logical errors that might only surface at runtime in traditional scripting are caught at compile time.

## The Implementation: Making Types Executable

Behind the scenes, Hypershell provides implementations that make these type-level constructs executable. The `Compare` handler demonstrates how parallel execution is achieved:

```rust
async fn handle(
    context: &Context,
    _tag: PhantomData<Compare<CodeA, CodeB>>,
    (input_a, input_b): (InputA, InputB),
) -> Result<bool, Context::Error> {
    let futures_a = context.handle(PhantomData::<CodeA>, input_a);
    let futures_b = context.handle(PhantomData::<CodeB>, input_b);

    let (res_a, res_b) = futures::try_join!(futures_a, futures_b)?;

    Ok(res_a == res_b)
}
```

This implementation creates two futures—one for each sub-program—and then uses `futures::try_join!` to execute them concurrently. This means that both URLs are downloaded and processed simultaneously, which is significantly more efficient than sequential execution. The results are then compared for equality using Rust's `Eq` trait.

The `If` handler provides the branching logic:

```rust
async fn handle(
    context: &Context,
    _tag: PhantomData<If<CodeCond, CodeThen, CodeElse>>,
    (input_cond, input_branch): (InputCond, InputBranch),
) -> Result<Output, Context::Error> {
    let cond = context.handle(PhantomData::<CodeCond>, input_cond).await?;

    if cond {
        context.handle(PhantomData::<CodeThen>, input_branch).await
    } else {
        context.handle(PhantomData::<CodeElse>, input_branch).await
    }
}
```

This handler first executes the condition sub-program and waits for its result. Based on whether that result is true or false, it then executes either the "then" or "else" branch. The entire process is asynchronous and composable with other Hypershell constructs.

## Context and Component Wiring

The example uses a custom `HypershellComparePreset` that extends the standard Hypershell capabilities with the new `Compare` and `If` handlers. This demonstrates Hypershell's modular architecture—new language constructs can be added by implementing the appropriate traits and wiring them into the component system.

```rust
#[cgp_context(MyAppComponents: HypershellComparePreset)]
#[derive(HasField)]
pub struct MyApp {
    pub http_client: Client,
    pub url_a: String,
    pub url_b: String,
}
```

The `MyApp` context provides the runtime values that the type-level program needs: an HTTP client for making requests and the two URLs to compare. The `#[derive(HasField)]` attribute allows the type-level `FieldArg<"url_a">` and `FieldArg<"url_b">` constructs to extract these values at runtime.

## Execution Flow and Input Handling

When the program runs, it requires a specific input structure:

```rust
app.handle(
    PhantomData::<Program>,
    ((<Vec<u8>>::new(), <Vec<u8>>::new()), Vec::new()),
).await?;
```

This input tuple structure reflects the nested nature of the program. The outer tuple provides inputs for the `If` construct: the first element is a tuple containing inputs for the two `Compare` sub-programs, and the second element is the input for whichever branch gets executed. Since the HTTP requests don't need input data, empty vectors are provided.

## The Power of Type-Level Composition

What makes this example remarkable is how it demonstrates the composability of type-level DSLs. The `GetChecksumOf` pipeline is defined once but used twice within the `Compare` construct. The `Compare` construct itself is embedded within the `If` construct, which is then piped to `StreamToStdout`. Each level of composition maintains type safety while enabling reuse and modularity.

This approach offers several advantages over traditional scripting approaches. First, the structure of the program is verified at compile time, preventing many runtime errors. Second, the parallel execution happens automatically based on the program structure—no explicit threading or async coordination is required. Third, the type-level abstraction makes the program's intent incredibly clear, even for complex control flows.

## Real-World Applications

While this example uses simple echo commands for the branches, the same pattern could be applied to much more complex scenarios. You could compare database query results and take different actions based on the outcome, or compare the output of different algorithms and choose the best result. The type-level approach scales naturally to arbitrarily complex conditional logic while maintaining compile-time verification and automatic parallelization.

This example showcases Hypershell's vision of making shell scripting both more powerful and more reliable through the application of advanced type-system features. By moving control flow to the type level, programs become more expressive, more efficient, and less prone to the kinds of errors that plague traditional shell scripts.
