# Reference

## Table of Contents

* [Overview](README.md)
* [Usage](Usage.md)
* [**Reference**](Reference.md)
  * [Target Format Reference](#target-format-reference)
    * [Source Layouts](#source-layouts)
    * [Test Target Layouts](#test-target-layouts)
    * [Other Rules](#other-rules)
  * [Package Manifest File Format Reference](#package-manifest-file-format-reference)
    * [Package Declaration](#package-declaration)
    * [Package](#package)
    * [Package Dependency](#package-dependency)
  * [Version](#version)
  * [Customizing Builds](#customizing-builds)
  * [Build Configurations](#build-configurations)
    * [Debug](#debug)
    * [Release](#release)
  * [Depending on Apple Modules](#depending-on-apple-modules)
  * [C language targets](#c-language-targets)
* [Resources](Resources.md)

---

## Target Format Reference

### Source Layouts

The targets that `swift build` creates are determined from the filesystem
layout of your source files.

For example, if you create a directory with the following layout:

    example/
    example/Sources/bar.swift
    example/Sources/baz.swift

This defines a single target, named after the package name from
`Package.swift`.

To create multiple targets, create multiple subdirectories:

    example/Sources/Foo/Widget.swift
    example/Sources/Bar/Bazzer.swift

This defines two targets: `Foo` and `Bar`.

To generate an executable target (instead of a library target), add a
`main.swift` file to that target’s directory:

    example/Sources/Foo/main.swift

Running `swift build` will now produce an executable output file: `example/.build/debug/Foo`.

The C language targets are laid out in a similar format. For e.g. a library
`Baz` can be created with following layout:

    example/Sources/Baz/Baz.c
    example/Sources/Baz/include/Baz.h

The public headers for this library go in the directory `include`.

Similarly, an executable C language target `Baz` looks like this:

    example/Sources/Baz/main.c

Note: It is possible to have C, C++, Objective-C and Objective-C++ sources as
part of a C language target. Swift targets can import C language targets but
not vice versa.

Read more on C language targets [here](#c-language-targets).

### Test Target Layouts

The package manager supports laying out test sources following a similar
convention as primary sources:

    example/Tests/FooTests/WidgetTests.swift

This defines a `FooTests` test target. By convention, when there is a target
`Foo` and a matching test target `FooTests`, the package manager will establish
an implicit dependency between the test target and the target it assumes it is
trying to test.

On Linux, the `XCTest` testing framework does not support dynamic discovery of
tests. Instead, packages which are intended for use on Linux should include an:

    example/Tests/LinuxMain.swift

file which imports all of the individual test targets in the package, and then
invokes `XCTest.XCTMain` passing it the list of all tests.

### Other Rules

* `Tests` or any other subdirectory can be [excluded](#exclude) via Manifest
  file.
* Subdirectories of a directory named `Sources`, `Source`, `srcs` or `src` in
  the root directory become target.
* It is acceptable to have no `Sources` directory, in which case the root
  directory is treated as a single target (place your sources there) or sub
  directories of the root are considered targets. Use this layout convention
  for simple projects.

---

## Package Manifest File Format Reference

Instructions for how to build a package are provided by the `Package.swift`
manifest file. `Package.swift` is a Swift file defining a single `Package`
object. This object is configured via the APIs defined in the
`PackageDescription` Swift target supplied with the Swift Package Manager.

### Package Declaration

Every `Package.swift` file should follow the following format:

```swift
import PackageDescription

/// The package description.
let package = Package(/* ... */)

// ... subsequent package configuration APIs can be used here to further
// configure the package ...
```

Conceptually, the description defined by the `Package.swift` file is _combined_
with the information on the package derived from the filesystem conventions
described previously.

### Package

```swift
Package(
    name: String,
    pkgConfig: String? = nil,
    providers: [SystemPackageProvider]? = nil,
    targets: [Target] = [],
    dependencies: [Package.Dependency] = [],
    swiftLanguageVersions: [Int]? = nil,
    exclude: [String] = []
)
```

\- [name](#name): The name of the package.  
\- [pkgConfig](#pkgconfig): Name of the pkg-config (.pc) file to get the
additional flags for system modules.  
\- [providers](#providers): Defines hints to display for installing system
modules.  
\- [targets](#targets): Additional information on each target.  
\- [dependencies](#dependencies): Declare dependencies on external packages.  
\- [swiftLanguageVersions](#swiftlanguageversions): Specifies the set of
supported Swift language versions.  
\- [exclude](#exclude): Exclude files and directories from package sources.  

Creates a new package instance. There should only be one package declared per
manifest. The parameters here supply the package description and are documented
in further detail below.

#### name

```swift
import PackageDescription

let package = Package(
    name: "FooBar"
)
```

This is the minimal requirement for a manifest to be valid. When the sources
are located directly under `Sources/` directory, there is only one target and
the target name will be the same as the package name.

#### pkgConfig

This property should only be used for System Module Packages. It defines the
name of the pkg-config (.pc) file that should be searched and read to get the
additional flags like include search path, linker search path, system libraries
to link etc.

```swift
import PackageDescription

let package = Package(
    name: "CGtk3",
    pkgConfig: "gtk+-3.0"
)
```

Here `gtk+-3.0.pc` will be searched in standard locations for the current
system. Users can provide their own paths for location of pc files using the
environment variable, `PKG_CONFIG_PATH`, which will be searched before the
standard locations.

_NOTE: This feature does not require pkg-config to be installed. However, if
installed it will used to find additional platform specific pc file locations
which might be unknown to SwiftPM._

#### providers

This property should only be used for system module packages. It can be used to
provide _hints_ for other users to install a System Module using
a system package manager like homebrew, apt-get etc.

_NOTE: SwiftPM will *never* execute the command and only suggest the users to
run it._

```swift
import PackageDescription

let package = Package(
    name: "CGtk3",
    pkgConfig: "gtk+-3.0",
    providers: [
        .Brew("gtk+3"),
        .Apt("gtk3")
    ]
)
```

In this case if SwiftPM determines that GTK 3 package is not installed, it will
output an appropriate hint depending on which platform the user is on i.e.
macOS, Ubuntu, etc.

#### targets

The targets property is required when you have more than one target in your
package and need to declare a dependency between them.

```swift
import PackageDescription

let package = Package(
    name: "Hello",
    targets: [
        Target(name: "Bar", dependencies: ["Foo"]),
    ]
)
```

The name identifies which target, the information is being associated with, and
the list of dependencies specifies the names of other targets in the same
package which must be built before that target. In the example here, `Foo` and
`Bar` are targets present under `Sources/` directory, and a dependency is being
establish on `Foo` from `Bar`. This will cause the `Foo` target to be built
before `Bar` target so that it can be imported:

_NOTE: It is also possible to declare target dependencies between a test and
regular target._

#### dependencies

This is the list of packages that the current package depends on and
information about the required versions. You can specify a URL (or local path)
to any valid Swift package.

```swift
import PackageDescription

let package = Package(
    name: "Example",
    dependencies: [
        .Package(url: "ssh://git@example.com/Greeter.git", versions: Version(1,0,0)..<Version(2,0,0)),
        .Package(url: "../StringExtensions", "1.0.0"),
        .Package(url: "https://github.com/MyAwesomePackage", majorVersion: 1, minor: 4),
    ]
)
```

This is a list of `Package.Dependency` instances, see [Package
Dependency](#package-dependency) for available options.

#### swiftLanguageVersions

This property is used to specify the set of supported Swift language versions.

The package manager will select the Swift language version that is most close
to (but not exceeding) the major version of the Swift compiler in use.  It is
an error if a package does not support any version compatible with the current
compiler. For e.g. if Swift language version is set to `[3]`, both Swift 3 and
4 compilers will select '3', and if Swift language version is set to `[3, 4]`,
Swift 3 compiler will select '3' and Swift 4 compiler will select '4'.

If a package does not specify any Swift language versions, the language version
to be used will match the major version of the package's [Swift tools
version](Usage.md#swift-tools-version).  For e.g.: A Swift tools version with a
major version of '3' will imply a default Swift language version of '3', and a
Swift tools version with a major version of '4' will imply a default Swift
language version of '4'.

#### exclude

Use this property to exclude files and directories from the package sources.

Every item specifies a relative path from the root of the package.

```swift
let package = Package(
    name: "Foo",
    exclude: ["Sources/Fixtures", "Sources/readme.md", "Tests/FooTests/images"]
)
```

This is helpful when you want to place files like resources or fixtures that
should not be considered by the convention system as possible sources.

### Package Dependency

A `Package.Dependency` represents the location and and required version
information of an external dependency. The version range controls what versions
of a package dependency are expected to work with the current package. When the
package manager is fetching the complete set of packages required to build a
package, it considers all of the version range specifications from all of the
packages in order to select appropriate versions.

```swift
.Package(url: String, versions: Range<Version>)
```
\- *url*: URL or local path to a Package.  
\- *versions*: The range of [versions](#version) which are required.  

```swift
.Package(url: String, versions: ClosedRange<Version>)
```
\- *url*: URL or local path to a Package.  
\- *versions*: The closed range of [versions](#version) which are required.  

```swift
.Package(url: String, majorVersion: Int)
```
\- *url*: URL or local path to a Package.  
\- *majorVersion*: The major version which is required.  

This is a short-hand form for specifying a range including all versions of a
major version, and is the recommended way for specifying a dependency following
the [semantic versioning](http://semver.org) standard.

```swift
.Package(url: String, majorVersion: Int, minor: Int)
```
\- *url*: URL or local path to a Package.  
\- *majorVersion*: Major version to consider.  
\- *minor*: Minor version to consider.  

As for the prior API, this is a short-hand form for specifying a range that
inclues all versions of a major and minor version.

```swift
.Package(url: String, _ version: Version)
```
\- *url*: URL or local path to a Package.  
\- *version*: The exact [Version](#version) which is required.  

## Version

A struct representing a [semantic version](http://semver.org).

```swift
Version(
	_ major: Int,
	_ minor: Int,
	_ patch: Int,
	prereleaseIdentifiers: [String] = [],
	buildMetadataIdentifier: String? = nil
)
```

\- *major*: The major version, incremented when you make incompatible API
changes.  
\- *minor*: The minor version, incremented when you add functionality in a
backwards-compatible manner.  
\- *patch*: The patch version, incremented when you make backwards-compatible
bug fixes.  
\- *prereleaseIdentifiers*: Used to denote a pre-released version for eg:
alpha, beta, etc.  
\- *buildMetadataIdentifier*: Optional build meta data for eg: timestamp, hash,
etc.  

A `Version` struct can be initialized using a string literal in following
format:

``` "major.minor.patch[-prereleaseIdentifiers][+buildMetadata]" ```

where `prereleaseIdentifiers` and `buildMetadata` are optional.  
_NOTE: prereleaseIdentifiers are separated by dot (.)._

### Customizing Builds

Using Swift as the format for the manifest allows for powerful customization,
for example:

```swift
import PackageDescription

var package = Package(name: "Example")

#if os(Linux)
let target = Target(name: "LinuxSources/foo")
package.targets.append(target)
#endif
```

## Build Configurations

SwiftPM allows two build configurations: Debug (default) and Release.

### Debug

By default, running `swift build` will build in debug configuration.
Alternatively, you can also use `swift build -c debug`. The build artifacts are
located in directory called `debug` under build folder.  A Swift target is
built with following flags in debug mode:  

* `-Onone`: Compile without any optimization.
* `-g`: Generate debug information.
* `-enable-testing`: Enable Swift compiler's testability feature.

A C language target is build with following flags in debug mode:

* `-O0`: Compile without any optimization.
* `-g`: Generate debug information.

### Release

To build in release mode, type: `swift build -c release`. The build artifacts
are located in directory called `release` under build folder.  A Swift target
is built with following flags in release mode:  

* `-O`: Compile with optimizations.
* `-whole-module-optimization`: Optimize input files (per module) together
  instead of individually.

A C language target is build with following flags in release mode:

* `-O2`: Compile with optimizations.

## Depending on Apple Modules

At this time there is no explicit support for depending on UIKit, AppKit,
etc, though importing these modules should work if they are present in the
proper system location. We will add explicit support for system dependencies in
the future. Note that at this time the Package Manager has no support for iOS,
watchOS, or tvOS platforms.

## C language targets

The C language targets are laid out similar to Swift targets execept that the C
langauge libraries should contain a directory named `include` to hold the
public headers.  
To allow a Swift target to import a C language target, add a [target
dependency](#targets) in the manifest file. Swift Package Manager will
automatically generate a modulemap for each C language library target for these
3 cases:

* If `include/Foo/Foo.h` exists and `Foo` is the only directory under the
  include directory then `include/Foo/Foo.h` becomes the umbrella header.

* If `include/Foo.h` exists and `include` contains no other subdirectory then
  `include/Foo.h` becomes the umbrella header.

* Otherwise if the `include` directory only contains header files and no other
  subdirectory, it becomes the umbrella directory.

In case of complicated `include` layouts, a custom `module.modulemap` can be
provided inside `include`. SwiftPM will error out if it can not generate a
modulemap w.r.t the above rules.

For executable targets, only one valid C language main file is allowed i.e. it
is invalid to have `main.c` and `main.cpp` in the same target.
