---
title: Handle errors in ASP.NET Core Blazor apps
author: guardrex
description: Discover how ASP.NET Core Blazor how Blazor manages unhandled exceptions and how to develop apps to detect and handle errors.
monikerRange: '>= aspnetcore-3.0'
ms.author: riande
ms.custom: mvc
ms.date: 07/20/2019
uid: blazor/handle-errors
---
# Handle errors in ASP.NET Core Blazor apps

By [Steve Sanderson](https://github.com/SteveSandersonMS) and [Luke Latham](https://github.com/guardrex)

This article describes how Blazor manages unhandled exceptions and how to develop apps to detect and handle errors.

## How the Blazor framework reacts to unhandled exceptions

Blazor server-side is a stateful framework. While users interact with an app, they maintain a connection to the server known as a *circuit*. The circuit holds active component instances, plus many other aspects of state, such as:

* The most recent rendered output of components.
* The current set of event-handling delegates that could be triggered by client-side events.

If a user opens the app in multiple browser tabs, they have multiple independent circuits.

Blazor treats most unhandled exceptions as fatal to the circuit where they occur. If a circuit is terminated due to an unhandled exception, the user can only continue to interact with the app by reloading the page to create a new circuit. Circuits outside of the one that's terminated, which are circuits for other users or other browser tabs, aren't affected. This scenario is similar to a desktop app that crashes&mdash;the crashed app must be restarted, but other apps aren't affected.

The reason for terminating a circuit for most unhandled exceptions is that an unhandled exception often leaves the circuit in an undefined state. If the framework continues processing a component's code after an unhandled exception and regardless of the circuit's state, the app's normal operation can't be guaranteed. Even security vulnerabilities in the app may result.

## Manage unhandled exceptions in developer code

If you want users to be able to continue using an app after an error occurs (for example, when loading or saving data), develop the app with error handling logic. C# [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statements are useful in error handling scenarios and are a suggested approach in later sections of this article that describe potential sources of unhandled exceptions.

The structure of a `try-catch` statement includes two *blocks* of code:

* The `try` block contains code that may throw an exception.
* The `catch` block contains code that executes when an exception occurs in the `try` block.

```csharp
try
{
    ...
}
catch (Exception ex)
{
    ...
}
```

In many C# development scenarios, trapping *all exceptions* with <xref:System.Exception?displayProperty=fullName> isn't considered a *best practice*. Usually, a more specific exception is trapped because:

* You should only catch those exceptions that you know how to recover from.
* Specific exceptions can be handled in a fine-grained manner:
  * Specific method calls and property settings can be made by the app for a specific error condition.
  * Specific error messages can be logged and shown to the user.

In Blazor exception scenarios, however, we recommend trapping all exceptions in components where unhandled exceptions may occur. Trapping all exceptions provides an opportunity to:

* Update the rendered component UI with a general exception message for the user.
* Log a general exception for debugging.

This approach usually avoids a circuit failure and allows a user to continue interacting with a component.

`try-catch` statements can be nested inside of other `try-catch` statements. When nested statements are used, more specific exceptions can be caught before the most general type of exception is trapped with `System.Exception`.

For more information on `try-catch` statements, including how to nest statements, see the [try-catch statement guidance in the C# language reference](/dotnet/csharp/language-reference/keywords/try-catch).

Be careful not to disclose sensitive information to end users. Under normal conditions in a production app, don't render framework exception messages or stack traces in the UI, as doing so may help a malicious user discover weaknesses in an app that can compromise the security of the app, server, or network.

## Log errors with a persistent provider

If an unhandled exception occurs, the exception is logged to <xref:Microsoft.Extensions.Logging.ILogger> instances configured in the service container. By default, template-based ASP.NET Core Blazor apps log to console output with the Console Logging Provider. Consider logging to a more permanent location with a provider that manages log size and log rotation. For more information, see <xref:fundamentals/logging/index>.

During development, Blazor usually sends the full details of exceptions to the browser's console to aid in debugging. In production, detailed errors in the browser's console are disabled by default, which means that errors aren't sent to clients but the exception's full details are still logged server-side. For more information, see <xref:fundamentals/error-handling>.

You must decide which incidents to log and the level of severity of logged incidents. Note that hostile users might be able to trigger errors deliberately (for example, by supplying an unknown `ProductId` in the URL of a component that displays product details), so you shouldn't necessarily treat all errors as high-severity incidents with logging.

## Places where errors may occur

Framework and app code may trigger unhandled exceptions in any of the following locations, which are described in the following sections of this article:

* [Component instantiation](#component-instantiation)
* [Lifecycle methods](#lifecycle-methods)
* [Rendering logic](#rendering-logic)
* [Event handlers](#event-handlers)
* [Component disposal](#component-disposal)
* [JavaScript interop](#javascript-interop)
* [Circuit handlers](#circuit-handlers)
* [Circuit disposal](#circuit-disposal)
* [Prerendering](#prerendering)

### Component instantiation

When Blazor creates an instance of a component, it invokes the component's constructor and the constructors for any DI services supplied to the component's constructor via the [@inject](xref:blazor/dependency-injection#request-a-service-in-a-component) directive or the [[Inject]](xref:blazor/dependency-injection#request-a-service-in-a-component) attribute. If any of the executed constructors throw an exception, or if a setter for any `[Inject]` properties throw an exception, it's fatal to the circuit because it's impossible for the framework to instantiate the component.

### Lifecycle methods

During the lifetime of a component, Blazor invokes lifecycle methods, such as `OnInitialized`, `OnParametersSet`, `ShouldRender`, `OnAfterRender`, and the asynchronous versions of these methods. If any lifecycle method throws an exception, synchronously or asynchronously, it's fatal to the circuit. For components to deal with errors in lifecycle methods, add suitable error handling logic to the method.

In the following example where `OnParametersSetAsync` calls a method to obtain a product:

* An exception thrown in the `ProductRepository.GetProductByIdAsync` method is handled by a `try`-`catch` statement.
* When the `catch` block is executed:
  * `loadFailed` is set to `true`, which is used to display an error message to the user.
  * The error is logged.

[!code-cshtml[](handle-errors/samples_snapshot/3.x/product-details.razor?highlight=11,27-39)]

### Rendering logic

The declarative markup in a `.razor` component file is compiled into a C# method called `BuildRenderTree`. When a component renders, `BuildRenderTree` executes and builds up a data structure describing the elements, text, and child components of the rendered component.

Rendering logic can throw an exception. An example of this scenario occurs when `@someObject.PropertyName` is evaluated but `@someObject` is `null`. An unhandled exception thrown by rendering logic is fatal to the circuit.

To prevent a null reference exception in rendering logic, check for a `null` object before accessing its members. In the following example, `person.Address` properties aren't accessed if `person.Address` is `null`:

[!code-cshtml[](handle-errors/samples_snapshot/3.x/person-example.razor?highlight=1)]

The preceding code example assumes that `@person` isn't `null`. Often, the structure of the code guarantees that an object exists at the time it's rendered. In those cases, it isn't necessary to check for `null` in rendering logic. In the prior example, `person` might be guaranteed to exist because it's created when the component is instantiated.

### Event handlers

When event handlers are created using `@onclick`, `@onchange`, and other `@on...` attributes, or `@bind` is used, client-side code can trigger invocations of C# code. Event handler code might throw an unhandled exception in these scenarios.

If an event handler throws an unhandled exception, it's fatal to the circuit. If the app calls code that could fail for external reasons (for example, a database query fails), the app should trap exceptions using a [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statement with error handling and logging. If user code doesn't trap and handle the exception, the framework logs the exception and terminates the circuit.

### Component disposal

When a component that implements <xref:System.IDisposable?displayProperty=fullName> is removed from the UI, for example because the user has navigated to another page, the framework calls the component's <xref:System.IDisposable.Dispose*> method.

If the component's `Dispose` method throws an unhandled exception, it's fatal to the circuit. If disposal logic may throw exceptions, the app should trap exceptions using a [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statement with error handling and logging.

For more information on component disposal, see <xref:blazor/components#component-disposal-with-idisposable>.

### JavaScript interop

`IJSRuntime.InvokeAsync<T>` allows .NET code to make asynchronous calls to the JavaScript runtime in the user's browser.

The following conditions apply to error handling with `InvokeAsync<T>`:

* If a call to `InvokeAsync<T>` fails synchronously, for example because the supplied arguments can't be serialized, a .NET exception occurs. It's up to the developer to provide code to catch the exception. If app code in an event handler or component lifecycle method doesn't handle an exception, the resulting exception is fatal to the circuit.
* If a call to `InvokeAsync<T>` fails asynchronously, for example because the JavaScript-side code throws an exception or returns a `Promise` that completed as `rejected`, the .NET <xref:System.Threading.Tasks.Task> fails. It's up to the developer to provide code to catch the exception. If using the [await](/dotnet/csharp/language-reference/keywords/await) operator, consider wrapping the method call in a [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statement with error handling and logging. Otherwise, the failing code results in an unhandled exception that's fatal to the circuit.
* By default, calls to `InvokeAsync<T>` must complete within a certain period or or else the call times out. The default timeout period is one minute. The timeout protects the code against a loss in network connectivity or JavaScript code that never sends back a completion message. If the call times out, the resulting `Task` fails with an <xref:System.OperationCanceledException>. Trap and process the exception with logging.

Similarly, it's possible for JavaScript code to initiate calls to .NET methods indicated by the [[JSInvokable] attribute](xref:blazor/javascript-interop#invoke-net-methods-from-javascript-functions). If these .NET methods throw an unhandled exception:

* It's *not* treated as fatal to the circuit.
* The JavaScript-side `Promise` is rejected.

You have the option of using error handling code on either the .NET side or the JavaScript side of the method call.

For more information, see <xref:blazor/javascript-interop>.

### Circuit handlers

Blazor allows code to define a *circuit handler*, which receives notifications when the state of a user's circuit changes. The following states are used:

* `initialized`
* `connected`
* `disconnected`
* `disposed`

Notifications are managed by registering a DI service that inherits from the `CircuitHandler` abstract base class.

If a custom circuit handler's methods throw an unhandled exception, it's fatal to the circuit. If app code must tolerate failures in handler code or any of a handler's called methods, wrap the code in one or more [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statements with error handling and logging.

### Circuit disposal

When a circuit ends because a user has disconnected and the framework is cleaning up the circuit state, the framework disposes of the circuit's DI scope. Disposing the scope disposes any circuit-scoped DI services that implement <xref:System.IDisposable?displayProperty=fullName>. If any DI service throws an unhandled exception during disposal, the framework logs the exception.

### Prerendering

Blazor components can be prerendered using `Html.RenderComponentAsync` so that their rendered HTML markup is returned as part of the user's initial HTTP request. This works by:

* Creating a new circuit containing all of the prerendered components that are part of the same page.
* Generating the initial HTML.
* Treating the circuit as `disconnected` until the user's browser establishes a SignalR connection back to the same server to resume interactivity on the circuit.

If any component throws an unhandled exception during prerendering, for example during a lifecycle method or in rendering logic, it's fatal to the circuit. Additionally, the exception is thrown up the call stack from the `Html.RenderComponentAsync` call, so the entire HTTP request fails unless the exception is explicitly caught by developer code.

Under normal circumstances when prerendering fails, continuing to build and render the component doesn't make sense given that a working component can't be rendered for the user. To tolerate errors that may occur during prerendering, error handling logic must be placed inside a component that may throw exceptions. Use [try-catch](/dotnet/csharp/language-reference/keywords/try-catch) statements with error handling and logging.

## Advanced scenarios

### Recursive rendering

Components can be nested recursively. This is useful for representing recursive data structures. For example, a `TreeNode` component can render more `TreeNode` components for each of the node's children.

When rendering recursively, avoid coding patterns that result in infinite recursion:

* Don't recursively render a data structure that contains a cycle (for example, a tree node whose children includes itself).
* Don't create a chain of layouts that contain a cycle (for example, a layout whose layout is itself).
* Don't allow an end user to violate recursion invariants (rules) through malicious data entry or JavaScript interop calls.

Infinite loops during rendering causes the rendering process to continue forever, which is equivalent to creating an unterminated loop. In these scenarios, the affected circuit hangs, and the thread usually attempts to:

* Consume as much CPU time as permitted by the operating system, indefinitely.
* Consume an unlimited amount of server memory, which is equivalent to the scenario where an unterminated loop adds entries to a collection on every iteration.

To avoid infinite recursion patterns, ensure that recursive rendering code contains suitable stopping conditions or enforces invariants (provide logic that imposes a strict specification on recursive sections of code to prevent such code from executing infinitely).

### Custom render tree logic

Most Blazor components are implemented as *.razor* files, which are compiled to produce logic that operates on a `RenderTreeBuilder` to render their output. It's also possible for a developer to manually implement `RenderTreeBuilder` logic using procedural C# code. For more information, see <xref:blazor/components#manual-rendertreebuilder-logic>.

> [!WARNING]
> Use of manual render tree builder logic is considered an advanced and unsafe scenario, not recommended for general component development.

If you choose to write such low-level code, it's your responsibility to guarantee the correctness of your code. For example, you must ensure that:

* Calls to `OpenElement` and `CloseElement` are correctly balanced.
* Attributes are only added in the correct places.

Incorrect manual render tree builder logic can cause arbitrary undefined behavior, including crashes, server hangs, and security vulnerabilities. Consider manual render tree builder logic on the same level of complexity and with the same level of *danger* as writing assembly code or MSIL instructions by hand.