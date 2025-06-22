# Prompt

Based on the blog post at [@2025-06-14-hypershell-release.md](@file:context-generic/contextgeneric.dev/content/blog/2025-06-14-hypershell-release.md) and the examples at [@hypershell-examples](@file:context-generic/hypershell/crates/hypershell-examples), give me an in-depth explanation of how the example [@compare_and_branch.rs](@file:context-generic/hypershell/crates/hypershell-examples/examples/compare_and_branch.rs) works.

Your post should be engaging, in full sentences, and avoid point forms. Your target audience is Rust developers who are new to CGP and Hypershell.

# Understanding Conditional Logic in Hypershell: A Deep Dive into the Compare and Branch Example

If you've been following along with Hypershell, you've seen how it elegantly handles streaming pipelines and shell command execution. But what happens when you need conditional logic in your pipelines? The `compare_and_branch.rs` example showcases an advanced pattern that extends Hypershell beyond simple linear pipelines into the realm of branching control flow.

## The Challenge: Comparing Remote Content

Imagine you need to compare two remote resources to see if they're identical. In a traditional shell script, you might download both files, compute their checksums, and use an if-statement to compare them. The `compare_and_branch.rs` example does exactly this, but in a type-safe, streaming fashion that leverages Hypershell's extensibility.

The program compares the SHA256 checksums of two URLs and prints a message indicating whether they're equal. What makes this interesting is that the entire operation happens at the type level, with the conditional branching baked into the program's type definition.

## Building Blocks: The Checksum Pipeline

Before diving into the branching logic, let's understand the foundational piece. The example defines a reusable type alias called `GetChecksumOf<Url>`:

```context-generic/hypershell/crates/hypershell-examples/examples/compare_and_branch.rs#L10-18
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

This type represents a pipeline that takes a URL as a type parameter and produces a hex-encoded SHA256 checksum. The pipeline streams the HTTP response directly into the checksum calculator, avoiding the need to buffer the entire response in memory. The `BytesToHex` handler then converts the raw checksum bytes into a human-readable hexadecimal string.

What's elegant here is that `GetChecksumOf` is parameterized by the URL type, allowing us to reuse this pattern for different URLs without duplicating code.

## The Conditional Logic: Compare and Branch

The main program introduces two new DSL constructs that aren't part of the standard Hypershell library: `Compare` and `If`. These are custom extensions that demonstrate how you can augment Hypershell with domain-specific control flow:

```context-generic/hypershell/crates/hypershell-examples/examples/compare_and_branch.rs#L20-40
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

The `Compare` construct takes two sub-programs (our checksum pipelines) and produces a boolean result. The `If` construct then uses this boolean to decide which branch to execute. If the checksums match, it runs the first `echo` command; otherwise, it runs the second.

## The Magic of Parallel Execution

One of the most impressive aspects of this implementation is what happens under the hood. When the `Compare` handler executes, it runs both checksum calculations in parallel. This is evident from looking at the `HandleCompare` implementation in the examples library, which uses `futures::try_join!` to await both operations concurrently.

This parallel execution is a natural consequence of Hypershell's async-first design. Since each pipeline is just an async computation, the framework can easily orchestrate multiple pipelines running simultaneously.

## Custom Contexts and Presets

To support these new constructs, the example uses a custom preset called `HypershellComparePreset`. This preset extends the standard `HypershellPreset` with additional handlers for the `Compare` and `If` operations:

```context-generic/hypershell/crates/hypershell-examples/examples/compare_and_branch.rs#L42-48
#[cgp_context(MyAppComponents: HypershellComparePreset)]
#[derive(HasField)]
pub struct MyApp {
    pub http_client: Client,
    pub url_a: String,
    pub url_b: String,
}
```

The `MyApp` context provides the two URLs to compare and the HTTP client needed for the requests. By using `HypershellComparePreset` instead of the standard preset, the context gains the ability to handle the custom DSL constructs.

## The Input Tuple Mystery

You might notice something unusual in the main function:

```context-generic/hypershell/crates/hypershell-examples/examples/compare_and_branch.rs#L59-61
app.handle(
    PhantomData::<Program>,
    ((<Vec<u8>>::new(), <Vec<u8>>::new()), Vec::new()),
)
```

Why is the input a nested tuple? This relates to how the branching logic works. The `If` handler needs two inputs: one for the condition and one for the branches. Since our condition (`Compare`) itself needs two inputs (one for each URL comparison), we end up with a structure of `((input_a, input_b), input_branch)`.

In this case, all inputs are empty vectors because the HTTP requests don't use the input stream—they fetch data from the URLs instead. But the type system still requires us to provide the correctly shaped input.

## Extending the Language

What makes this example particularly powerful is that it shows how Hypershell isn't just a fixed DSL—it's a framework for building DSLs. The `Compare` and `If` constructs aren't built into Hypershell; they're extensions added by the example library. This demonstrates the true power of Context-Generic Programming: you can extend the language with new constructs that feel native and compose seamlessly with existing ones.

The implementation leverages Rust's type system to ensure that all branches of the `If` statement produce compatible output types. If you tried to have one branch output a string and another output a number, the compiler would catch this error at compile time.

## Real-World Applications

This pattern of comparing remote resources and branching based on the result has numerous practical applications. You could use it to verify mirror synchronization, detect when a remote resource has changed, or implement conditional deployment pipelines. The streaming nature means you can compare large files without loading them entirely into memory, and the parallel execution ensures optimal performance.

## Conclusion

The `compare_and_branch.rs` example showcases Hypershell's extensibility and the power of type-level DSLs in Rust. By encoding control flow in types, we get compile-time verification of our branching logic while maintaining the expressiveness needed for real-world applications. The example demonstrates that Hypershell isn't just about running shell commands—it's about building type-safe, composable, and extensible pipeline abstractions that can handle complex control flow patterns.

For developers new to CGP and Hypershell, this example illustrates an important principle: the framework provides not just functionality, but a foundation for building your own domain-specific abstractions. The same techniques used to implement `Compare` and `If` can be applied to create custom handlers for your specific use cases, all while maintaining type safety and composability.
