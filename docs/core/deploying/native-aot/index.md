---
title: Native AOT deployment overview
description: Learn what native AOT deployments are and why you should consider using it as part of the publishing your app with .NET 7 and later.
author: lakshanf
ms.author: lakshanf
ms.date: 05/05/2023
---
# Native AOT deployment

Publishing your app as *native AOT* produces an app that's [self-contained](../index.md#publish-self-contained) and that has been ahead-of-time (AOT) compiled to native code. Native AOT apps have faster startup time and smaller memory footprints. These apps can run on machines that don't have the .NET runtime installed.

The benefit of native AOT is most significant for workloads with a high number of deployed instances, such as cloud infrastructure and hyper-scale services. Native AOT deployment is currently in [preview for ASP.NET Core 8.0](/aspnet/core/fundamentals/native-aot/?view=aspnetcore-8.0&preserve-view=true).

The native AOT deployment model uses an ahead-of-time compiler to compile IL to native code at the time of publish. Native AOT apps don't use a just-in-time (JIT) compiler when the application runs. Native AOT apps can run in restricted environments where a JIT is not allowed. Native AOT applications target a specific runtime environment, such as Linux x64 or Windows x64, just like publishing a [self-contained app](../index.md#publish-self-contained).

There are some limitations in the .NET native AOT deployment model, with the main one being that run-time code generation isn't possible. For more information, see [Limitations of Native AOT deployment](#limitations-of-native-aot-deployment). The support in the .NET 7 release is targeted towards console-type applications.

> [!WARNING]
> In .NET 7, only a limited number of libraries are fully compatible with native AOT.

## Prerequisites

The following prerequisites need to be installed before publishing .NET projects with native AOT.

On Windows, install [Visual Studio 2022](https://visualstudio.microsoft.com/vs/), including the **Desktop development with C++** workload with all default components.

On Linux, install the compiler toolchain and developer packages for libraries that the .NET runtime depends on.

- Ubuntu (18.04+)

  ```sh
  sudo apt-get install clang zlib1g-dev
  ```

- Alpine (3.15+)

  ```sh
  sudo apk add clang build-base zlib-dev
  ```

On macOS, install the latest [Command Line Tools for XCode](https://developer.apple.com/xcode/download/). Supported on .NET 8 and later versions.

## Publish native AOT using the CLI

01. Add `<PublishAot>true</PublishAot>` to your project file.

    This property enables native AOT compilation during publish. It also enables dynamic code-usage analysis during build and editing. It's preferable to put this setting in the project file rather than passing it on the command line, since it controls behaviors outside publish.

    ```xml
    <PropertyGroup>
        <PublishAot>true</PublishAot>
    </PropertyGroup>
    ```

02. Publish the app for a specific runtime identifier using `dotnet publish -r <RID>`.

    The following example publishes the app for Windows as a native AOT application on a machine with the required prerequisites installed.

    `dotnet publish -r win-x64 -c Release`

    The following example publishes the app for Linux as a native AOT application. A native AOT binary produced on Linux machine is only going to work on same or newer Linux version. For example, native AOT binary produced on Ubuntu 20.04 is going to run on Ubuntu 20.04 and later, but it is not going to run on Ubuntu 18.04.

    `dotnet publish -r linux-arm64 -c Release`

The app will be available in the publish directory and will contain all the code needed to run in it, including a stripped-down version of the coreclr runtime.

Check out the [native AOT samples](https://github.com/dotnet/samples/tree/main/core/nativeaot) available in the dotnet/samples repository on GitHub. The samples include [Linux](https://github.com/dotnet/samples/blob/main/core/nativeaot/HelloWorld/Dockerfile) and [Windows](https://github.com/dotnet/samples/blob/main/core/nativeaot/HelloWorld/Dockerfile.windowsservercore-x64) Dockerfiles that demonstrate how to automate installation of prerequisites and publish .NET projects with native AOT using containers.

## Native debug information

### [.NET 7](#tab/net7)

Native AOT publishing follows platform conventions for native toolchains. The default behavior of native toolchains on Windows is to produce debug information in a separate *.pdb* file. The default behavior of native toolchains on Linux is to include the debug information in the native binary, which makes the native binary significantly larger.

Set the `StripSymbols` property to `true` to produce the debug information in a separate *.dbg* file and exclude it from the native binary on Linux. (This property has no effect on Windows.)

```xml
<PropertyGroup>
    <StripSymbols>true</StripSymbols>
</PropertyGroup>
```

### [.NET 8+](#tab/net8plus)

By default, native AOT publishing produces debug information in a separate file on all supported platforms (*.dbg* on Linux, *.pdb* on Windows, and *.dwarf* on macOS). This file is necessary for running the app under the debugger or inspecting crash dumps. On Unix-like platforms, you can set the `StripSymbols` property to `false` to include the debug information in the native binary. (Including the debug information makes the native binary significantly larger.)

```xml
<PropertyGroup>
    <StripSymbols>false</StripSymbols>
</PropertyGroup>
```

---

## Limitations of native AOT deployment

Native AOT applications come with a few fundamental limitations and compatibility issues. The key limitations include:

- No dynamic loading (for example, `Assembly.LoadFile`).
- No run-time code generation (for example, `System.Reflection.Emit`).
- No C++/CLI.
- No built-in COM (only applies to Windows).
- Requires trimming, which has [limitations](../trimming/incompatibilities.md).
- Implies compilation into a single file, which has known [incompatibilities](../single-file/overview.md#api-incompatibility).
- Apps include required runtime libraries (just like [self-contained apps](../index.md#publish-self-contained), increasing their size as compared to framework-dependent apps).
- <xref:System.Linq.Expressions> always use interpreted form, that is much slower than run-time generated compiled code.

The publish process analyzes the entire project and its dependencies and produces warnings whenever the limitations could potentially be hit by the published application at run time.

The first release of native AOT in .NET 7 has these additional limitations:

- Should be targeted for console type applications (not ASP.NET Core).
- Not all the runtime libraries are fully annotated to be native AOT compatible (that is, some warnings in the runtime libraries are not actionable by end developers).
- Limited diagnostic support (debugging and profiling).

## Build native libraries

Publishing .NET class libraries as native AOT allows creating libraries that can be consumed from non-.NET programming languages. The produced native library is self-contained and doesn't require a .NET runtime to be installed.

Publishing a class library as native AOT creates a native library that exposes methods of the class library annotated with <xref:System.Runtime.InteropServices.UnmanagedCallersOnlyAttribute> with a non-null `EntryPoint` field. For more information, see the [native library sample](https://github.com/dotnet/samples/tree/main/core/nativeaot/NativeLibrary) available in the dotnet/samples repository on GitHub.

## Platform/architecture restrictions

The following table shows supported compilation targets.

| Platform | Supported architecture |
| -------- | ---------------------- |
| Windows  | x64, Arm64             |
| Linux    | x64, Arm64             |
| macOS*   | x64, Arm64             |

\* Supported starting in .NET 8.
