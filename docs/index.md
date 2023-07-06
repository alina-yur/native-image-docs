# GraalVM Native Image

# Collect Metadata with the Tracing Agent

The Native Image tool relies on the static analysis of an application's reachable code at runtime. 
However, the analysis cannot always completely predict all usages of the Java Native Interface (JNI), Java Reflection, Dynamic Proxy objects, or class path resources. 
Undetected usages of these dynamic features must be provided to the `native-image` tool in the form of [metadata](ReachabilityMetadata.md) (precomputed in code or as JSON configuration files).

Here you will find information how to automatically collect metadata for an application and write JSON configuration files.
To learn how to compute dynamic feature calls in code, see [Reachability Metadata](ReachabilityMetadata.md#computing-metadata-in-code).

### Table of Contents

* [Tracing Agent](#tracing-agent)
* [Conditional Metadata Collection](#conditional-metadata-collection)
* [Agent Advanced Usage](#agent-advanced-usage)
* [Native Image Configure Tool](#native-image-configure-tool)

## Tracing Agent

GraalVM provides a **Tracing Agent** to easily gather metadata and prepare configuration files. 
The agent tracks all usages of dynamic features during application execution on a regular Java VM.

Enable the agent on the command line with the `java` command from the GraalVM JDK:
```shell
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=/path/to/config-dir/ ...
```

>Note: `-agentlib` must be specified _before_ a `-jar` option or a class name or any application parameters as part of the `java` command.

When run, the agent looks up classes, methods, fields, resources for which the `native-image` tool needs additional information.
When the application completes and the JVM exits, the agent writes metadata to JSON files in the specified output directory (`/path/to/config-dir/`).

It may be necessary to run the application more than once (with different execution paths) for improved coverage of dynamic features.
The `config-merge-dir` option adds to an existing set of configuration files, as follows:
```shell
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-merge-dir=/path/to/config-dir/ ...                                                              ^^^^^
```

The agent also provides the following options to write metadata on a periodic basis:
- `config-write-period-secs=n`: writes metadata files every `n` seconds; `n` must be greater than 0.
- `config-write-initial-delay-secs=n`: waits `n` seconds before first writing metadata; defaults to `1`.

For example:
```shell
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=/path/to/config-dir/,config-write-period-secs=300,config-write-initial-delay-secs=5 ...
```

The above command will write metadata files to `/path/to/config-dir/` every 300 seconds after an initial delay of 5 seconds.

It is advisable to manually review the generated configuration files.
Because the agent observes only executed code, the application input should cover as many code paths as possible.

The generated configuration files can be supplied to the `native-image` tool by placing them in a `META-INF/native-image/` directory on the class path. 
This directory (or any of its subdirectories) is searched for files with the names `jni-config.json`, `reflect-config.json`, `proxy-config.json`, `resource-config.json`, `predefined-classes-config.json`, `serialization-config.json` which are then automatically included in the build process. 
Not all of those files must be present. 
When multiple files with the same name are found, all of them are considered.

To test the agent collecting metadata on an example application, go to the [Build a Native Executable with Reflection](guides/build-with-reflection.md) guide.

## Conditional Metadata Collection

The agent can deduce metadata conditions based on their usage in executed code.
Conditional metadata is mainly aimed towards library maintainers with the goal of reducing overall footprint.

To collect conditional metadata with the agent, see [Conditional Metadata Collection](ExperimentalAgentOptions.md#generating-conditional-configuration-using-the-agent).

## Agent Advanced Usage

### Caller-based Filters

By default, the agent filters dynamic accesses which Native Image supports without configuration.
The filter mechanism works by identifying the Java method performing the access, also referred to as _caller_ method, and matching its declaring class against a sequence of filter rules.
The built-in filter rules exclude dynamic accesses which originate in the JVM, or in parts of a Java class library directly supported by Native Image (such as `java.nio`) from the generated configuration files.
Which item (class, method, field, resource, etc.) is being accessed is not relevant for filtering.

In addition to the built-in filter, custom filter files with additional rules can be specified using the `caller-filter-file` option.
For example: `-agentlib:caller-filter-file=/path/to/filter-file,config-output-dir=...`

Filter files have the following structure:
```json
{ "rules": [
    {"excludeClasses": "com.oracle.svm.**"},
    {"includeClasses": "com.oracle.svm.tutorial.*"},
    {"excludeClasses": "com.oracle.svm.tutorial.HostedHelper"}
  ],
  "regexRules": [
    {"includeClasses": ".*"},
    {"excludeClasses": ".*\\$\\$Generated[0-9]+"}
  ]
}
```

The `rules` section contains a sequence of rules.
Each rule specifies either `includeClasses`, which means that lookups originating in matching classes will be included in the resulting configuration, or `excludeClasses`, which excludes lookups originating in matching classes from the configuration.
Each rule defines a pattern to match classes. The pattern can end in `.*` or `.**`, interpreted as follows:
    - `.*` matches all classes in the package and only that package;
    - `.**` matches all classes in the package as well as in all subpackages at any depth. 
Without `.*` or `.**`, the rule applies only to a single class with the qualified name that matches the pattern.
All rules are processed in the sequence in which they are specified, so later rules can partially or entirely override earlier ones.
When multiple filter files are provided (by specifying multiple `caller-filter-file` options), their rules are chained together in the order in which the files are specified.
The rules of the built-in caller filter are always processed first, so they can be overridden in custom filter files.

In the example above, the first rule excludes lookups originating in all classes from package `com.oracle.svm` and from all of its subpackages (and their subpackages, etc.) from the generated metadata.
In the next rule however, lookups from those classes that are directly in package `com.oracle.svm.tutorial` are included again.
Finally, lookups from the `HostedHelper` class is excluded again. Each of these rules partially overrides the previous ones.
For example, if the rules were in the reverse order, the exclusion of `com.oracle.svm.**` would be the last rule and would override all other rules.

The `regexRules` section also contains a sequence of rules.
Its structure is the same as that of the `rules` section, but rules are specified as regular expression patterns which are matched against the entire fully qualified class identifier.
The `regexRules` section is optional.
If a `regexRules` section is specified, a class will be considered included if (and only if) both `rules` and `regexRules` include the class and neither of them exclude it.
With no `regexRules` section, only the `rules` section determines whether a class is included or excluded.

For testing purposes, the built-in filter for Java class library lookups can be disabled by adding the `no-builtin-caller-filter` option, but the resulting metadata files are generally unsuitable for the build.
Similarly, the built-in filter for Java VM-internal accesses based on heuristics can be disabled with `no-builtin-heuristic-filter` and will also generally lead to less usable metadata files.
For example: `-agentlib:native-image-agent=no-builtin-caller-filter,no-builtin-heuristic-filter,config-output-dir=...`

### Access Filters

Unlike the caller-based filters described above, which filter dynamic accesses based on where they originate, _access filters_ apply to the _target_ of the access.
Therefore, access filters enable directly excluding packages and classes (and their members) from the generated configuration.

By default, all accessed classes (which also pass the caller-based filters and the built-in filters) are included in the generated configuration.
Using the `access-filter-file` option, a custom filter file that follows the file structure described above can be added.
The option can be specified more than once to add multiple filter files and can be combined with the other filter options, for example, `-agentlib:access-filter-file=/path/to/access-filter-file,caller-filter-file=/path/to/caller-filter-file,config-output-dir=...`.

### Specify Configuration Files as Arguments

A directory containing configuration files that is not part of the class path can be specified to `native-image` via `-H:ConfigurationFileDirectories=/path/to/config-dir/`.
This directory must directly contain all files: `jni-config.json`, `reflect-config.json`, `proxy-config.json` and `resource-config.json`.
A directory with the same metadata files that is on the class path, but not in `META-INF/native-image/`, can be provided via `-H:ConfigurationResourceRoots=path/to/resources/`.
Both `-H:ConfigurationFileDirectories` and `-H:ConfigurationResourceRoots` can also take a comma-separated list of directories.

### Injecting the Agent via the Process Environment

Altering the `java` command line to inject the agent can prove to be difficult if the Java process is launched by an application or script file, or if Java is even embedded in an existing process.
In that case, it is also possible to inject the agent via the `JAVA_TOOL_OPTIONS` environment variable.
This environment variable can be picked up by multiple Java processes which run at the same time, in which case each agent must write to a separate output directory with `config-output-dir`.
(The next section describes how to merge sets of configuration files.)
In order to use separate paths with a single global `JAVA_TOOL_OPTIONS` variable, the agent's output path options support placeholders:
```shell
export JAVA_TOOL_OPTIONS="-agentlib:native-image-agent=config-output-dir=/path/to/config-output-dir-{pid}-{datetime}/"
```

The `{pid}` placeholder is replaced with the process identifier, while `{datetime}` is replaced with the system date and time in UTC, formatted according to ISO 8601. 
For the above example, the resulting path could be: `/path/to/config-output-dir-31415-20181231T235950Z/`.

### Trace Files

In the examples above, `native-image-agent` has been used to both keep track of the dynamic accesses on a JVM and then to generate a set of configuration files from them.
However, for a better understanding of the execution, the agent can also write a _trace file_ in JSON format that contains each individual access:
```shell
$JAVA_HOME/bin/java -agentlib:native-image-agent=trace-output=/path/to/trace-file.json ...
```

The `native-image-configure` tool can transform trace files to configuration files.
The following command reads and processes `trace-file.json` and generates a set of configuration files in the directory `/path/to/config-dir/`:
```shell
native-image-configure generate --trace-input=/path/to/trace-file.json --output-dir=/path/to/config-dir/
```

### Interoperability

The agent uses the JVM Tool Interface (JVMTI) and can potentially be used with other JVMs that support JVMTI.
In this case, it is necessary to provide the absolute path of the agent:
```shell
/path/to/some/java -agentpath:/path/to/graalvm/jre/lib/amd64/libnative-image-agent.so=<options> ...
```

### Experimental Options

The agent has options which are currently experimental and might be enabled in future releases, but can also be changed or removed entirely.
See the [ExperimentalAgentOptions.md](ExperimentalAgentOptions.md) guide.

## Native Image Configure Tool

When using the agent in multiple processes at the same time as described in the previous section, `config-output-dir` is a safe option, but it results in multiple sets of configuration files.
The `native-image-configure` tool can be used to merge these configuration files.
This tool must first be built with:
```shell
native-image --macro:native-image-configure-launcher
```

Then, the tool can be used to merge sets of configuration files as follows:
```shell
native-image-configure generate --input-dir=/path/to/config-dir-0/ --input-dir=/path/to/config-dir-1/ --output-dir=/path/to/merged-config-dir/
```

This command reads one set of configuration files from `/path/to/config-dir-0/` and another from `/path/to/config-dir-1/` and then writes a set of configuration files that contains both of their information to `/path/to/merged-config-dir/`.
An arbitrary number of `--input-dir` arguments with sets of configuration files can be specified. See `native-image-configure help` for all options.
---

# Native Image Build Configuration

Native Image supports a wide range of options to configure the `native-image` builder.

### Table of Contents

* [Embed a Configuration File](#embed-a-configuration-file)
* [Configuration File Format](#configuration-file-format)
* [Order of Arguments Evaluation](#order-of-arguments-evaluation)
* [Memory Configuration for Native Image Build](#memory-configuration-for-native-image-build)
* [Specify Types Required to Be Defined at Build Time](#specify-types-required-to-be-defined-at-build-time)
 
## Embed a Configuration File

We recommend that you provide the configuration for the `native-image` builder by embedding a _native-image.properties_ file into a project JAR file.
The `native-image` builder will also automatically pick up all configuration options provided in the _META-INF/native-image/_ directory (or any of its subdirectories) and use it to construct `native-image` command-line options.

To avoid a situation when constituent parts of a project are built with overlapping configurations, we recommended you use subdirectories within _META-INF/native-image_: a JAR file built from multiple maven projects cannot suffer from overlapping `native-image` configurations.
For example:
* _foo.jar_ has its configurations in _META-INF/native-image/foo_groupID/foo_artifactID_
* _bar.jar_ has its configurations in _META-INF/native-image/bar_groupID/bar_artifactID_

The JAR file that contains `foo` and `bar` will then contain both configurations without conflict.
Therefore the recommended layout to store configuration data in JAR files is as follows:
```
META-INF/
└── native-image
    └── groupID
        └── artifactID
            └── native-image.properties
```

Note that the use of `${.}` in a _native-image.properties_ file expands to the resource location that contains that exact configuration file.
This can be useful if the _native-image.properties_ file refers to resources within its subdirectory, for example, `-H:ResourceConfigurationResources=${.}/custom_resources.json`.
Always make sure you use the option variants that take resources, that is, use `-H:ResourceConfigurationResources` instead of `-H:ResourceConfigurationFiles`.
Other options that work in this context are:
* `-H:DynamicProxyConfigurationResources`
* `-H:JNIConfigurationResources`
* `-H:ReflectionConfigurationResources`
* `-H:ResourceConfigurationResources`
* `-H:SerializationConfigurationResources`

By having such a composable _native-image.properties_ file, building a native executable does not require any additional option on the command line.
It is sufficient to run the following command:
```shell
$JAVA_HOME/bin/native-image -jar target/<name>.jar
```

To identify which configuration is applied when building a native executable, use `native-image --verbose`.
This shows from where `native-image` picks up the configurations to construct the final composite configuration command-line options for the native image builder.
```shell
native-image --verbose -jar build/basic-app-0.1-all.jar
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/common/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/buffer/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/transport/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/handler/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/codec-http/native-image.properties
...
Executing [
    <composite configuration command line options for the image builder>
]
```

Typical examples of configurations that use a configuration from _META-INF/native-image_ can be found in [Native Image configuration examples](https://github.com/graalvm/graalvm-demos/tree/master/native-image-configure-examples).

## Configuration File Format

A _native-image.properties_ file is a Java properties file that specifies configurations for `native-image`. 
The following properties are supported.

**Args**

Use this property if your project requires custom `native-image` command-line options to build correctly.
For example, the `native-image-configure-examples/configure-at-runtime-example` contains `Args = --initialize-at-build-time=com.fasterxml.jackson.annotation.JsonProperty$Access` in its _native-image.properties_ file to ensure the class `com.fasterxml.jackson.annotation.JsonProperty$Access` is initialized at executable build time.

**JavaArgs**

Sometimes it can be necessary to provide custom options to the JVM that runs the `native-image` builder.
Use the `JavaArgs` property in this case.

**ImageName**

This property specifies a user-defined name for the executable.
If `ImageName` is not used, a name is automatically chosen:
    * `native-image -jar <name.jar>` has a default executable name `<name>`
    * `native-image -cp ... fully.qualified.MainClass` has a default executable name `fully.qualified.mainclass`

Note that using `ImageName` does not prevent you from overriding the name via the command line.
For example, if `foo.bar` contains `ImageName=foo_app`:
    * `native-image -jar foo.bar` generates the executable `foo_app` but
    * `native-image -jar foo.bar application` generates the executable `application`

### Changing the Default Configuration Directory

Native Image by default stores configuration information in the user's home directory: _$HOME/.native-image/_.
To change this default, set the environment variable `NATIVE_IMAGE_USER_HOME` to a different location. For example:
```shell
export NATIVE_IMAGE_USER_HOME= $HOME/.local/share/native-image
```

## Order of Arguments Evaluation
The options passed to `native-image` are evaluated from left to right.
This also extends to options that are passed indirectly via configuration files in the _META-INF/native-image_ directory.
Consider the example where there is a JAR file that includes _native-image.properties_ containing `Args = -H:Optimize=0`.
You can override the setting that is contained in the JAR file by using the `-H:Optimize=2` option after `-cp <jar-file>`.

## Memory Configuration for Native Image Build

The `native-image` builder runs on a JVM and uses the memory management of the underlying platform.
The usual Java command-line options for garbage collection apply to the `native-image` builder.

During the creation of a native executable, the representation of the whole application is created to determine which classes and methods will be used at runtime.
It is a computationally intensive process that uses the following default values for memory usage:
```
-Xss10M \
-XX:MaxRAMPercentage=<percentage based on available memory> \
-XX:GCTimeRatio=19 \
-XX:+ExitOnOutOfMemoryError \
```
These defaults can be changed by passing `-J + <jvm option for memory>` to the `native-image` tool.

The `-XX:MaxRAMPercentage` value determines the maximum heap size of the builder and is computed based on available memory of the system.
It maxes out at 32GB by default and can be overwritten with, for example, `-J-XX:MaxRAMPercentage=90.0` for 90% of physical memory or `-Xmx4g` for 4GB.
`-XX:GCTimeRatio=19` increases the goal of the total time for garbage collection to 5%, which is more throughput-oriented and reduces peak RSS.
The build process also exits on the first `OutOfMemoryError` (`-XX:+ExitOnOutOfMemoryError`) to provide faster feedback in environments under a lot of memory pressure.

By default, the `native-image` tool uses up to 32 threads (but not more than the number of processors available). For custom values, use the `--parallelism=...` option.

For other related options available to the `native-image` tool, see the output from the command `native-image --expert-options-all`.

## Specify Types Required to Be Defined at Build Time

A well-structured library or application should handle linking of Java types (ensuring all reachable Java types are fully defined at build time) when building a native binary by itself.
The default behavior is to throw linking errors, if they occur, at runtime. 
However, you can prevent unwanted linking errors by specifying which classes are required to be fully linked at build time.
For that, use the `--link-at-build-time` option. 
If the option is used in the right context (see below), you can specify required classes to link at build time without explicitly listing classes and packages.
It is designed in a way that libraries can only configure their own classes, to avoid any side effects on other libraries.
You can pass the option to the `native-image` tool on the command line, embed it in a `native-image.properties` file on the module-path or the classpath.

Depending on how and where the option is used it behaves differently:

* If you use `--link-at-build-time` without arguments, all classes in the scope are required to be fully defined. If used without arguments on command line, all classes will be treated as "link-at-build-time" classes. If used without arguments embedded in a `native-image.properties` file on the module-path, all classes of the module will be treated as "link-at-build-time" classes. If you use `--link-at-build-time` embedded in a `native-image.properties` file on the classpath, the following error will be thrown:
    ```shell
    Error: Using '--link-at-build-time' without args only allowed on module-path. 'META-INF/native-image/org.mylibrary/native-image.properties' in 'file:///home/test/myapp/MyLibrary.jar' not part of module-path.
    ```
* If you use the  `--link-at-build-time` option with arguments, for example, `--link-at-build-time=foo.bar.Foobar,demo.myLibrary.Name,...`, the arguments should be fully qualified class names or package names. When used on the module-path or classpath (embedded in `native-image.properties` files), only classes and packages defined in the same JAR file can be specified. Packages for libraries used on the classpath need to be listed explicitly. To make this process easy, use the `@<prop-values-file>` syntax to generate a package list (or a class list) in a separate file automatically.

Another handy option is `--link-at-build-time-paths` which allows to specify which classes are required to be fully defined at build time by other means.
This variant requires arguments that are of the same type as the arguments passed via `-p` (`--module-path`) or `-cp` (`--class-path`):

```shell
--link-at-build-time-paths <class search path of directories and zip/jar files>
```

The given entries are searched and all classes inside are registered as `--link-at-build-time` classes.
This option is only allowed to be used on command line.

### Related Documentation

- [Class Initialization in Native Image](ClassInitialization.md)
- [Native Image Basics](NativeImageBasics.md)
- [Native Image Build Options](BuildOptions.md)
- [Native Image Build Overview](BuildOverview.md)
- [Reachability Metadata](ReachabilityMetadata.md)

#  Native Image Build Options

Depending on the GraalVM version, the options to the `native-image` builder may differ.

* `-cp, -classpath, --class-path <class search path of directories and zip/jar files>`: a `:` (`;` on Windows) separated list of directories, JAR archives, and ZIP archives to search for class files
* `-p <module path>, --module-path <module path>`: a `:` (`;` on Windows) separated list of directories. Each directory is a directory of modules.
* `--add-modules <module name>[,<module name>...]`: add root modules to resolve in addition to the initial module. `<module name>` can also be `ALL-DEFAULT`, `ALL-SYSTEM`, `ALL-MODULE-PATH`.
* `-D<name>=<value>`: set a system property 
* `-J<flag>`: pass an option directly to the JVM running the `native-image` builder
* `--diagnostics-mode`: enable diagnostics output which includes class initialization, substitutions, etc.
* `--enable-preview`: allow classes to depend on preview features of this release
* `--verbose`: enable verbose output
* `--version`: print the product version and exit
* `--help`: print this help message
* `--help-extra`: print help on non-standard options
* `--auto-fallback`: build standalone native executable if possible
* `--configure-reflection-metadata`: enable runtime instantiation of reflection objects for non-invoked methods
* `--enable-all-security-services`: add all security service classes to a generated native executable
* `--enable-http`: enable HTTP support in a native executable
* `--enable-https`: enable HTTPS support in a native executable
* `--enable-monitoring`: enable monitoring features that allow the VM to be inspected at run time. A comma-separated list can contain `heapdump`, `jfr`, `jvmstat`, `jmxserver` (experimental), `jmxclient` (experimental), or `all` (deprecated behavior: defaults to `all` if no argument is provided). For example: `--enable-monitoring=heapdump,jfr`.
* `--enable-sbom`: embed a Software Bill of Materials (SBOM) in a native executable or shared library for passive inspection. A comma-separated list can contain `cyclonedx`, `strict` (defaults to `cyclonedx` if no argument is provided), or `export` to save the SBOM to the native executable's output directory. The optional `strict` flag aborts the build if any class cannot be matched to a library in the SBOM. For example: `--enable-sbom=cyclonedx,strict`. (Not available in GraalVM Community Edition.)
* `--enable-url-protocols`: list comma-separated URL protocols to enable
* `--features`: a comma-separated list of fully qualified [Feature implementation classes](https://www.graalvm.org/sdk/javadoc/index.html?org/graalvm/nativeimage/hosted/Feature.html)
* `--force-fallback`: force building of a fallback native executable
* `--gc=<value>`: select Native Image garbage collector implementation. Allowed options for `<value>` are: `G1` for G1 garbage collector (not available in GraalVM Community Edition); `epsilon` for Epsilon garbage collector; `serial` for Serial garbage collector (default).
* `--initialize-at-build-time`: a comma-separated list of packages and classes (and implicitly all of their superclasses) that are initialized during generation of a native executable. An empty string designates all packages.
* `--initialize-at-run-time`: a comma-separated list of packages and classes (and implicitly all of their subclasses) that must be initialized at run time and not during generation. An empty string is currently not supported.
* `--install-exit-handlers`: provide `java.lang.Terminator` exit handlers
* `--libc`: select the `libc` implementation to use. Available implementations are `glibc`, `musl`, `bionic`.
* `--link-at-build-time`: require types to be fully defined at executable's build time. If used without arguments, all classes in scope of the option are required to be fully defined.
* `--link-at-build-time-paths`: require all types in given class or module-path entries to be fully defined at at executable's build time
* `--list-cpu-features`: show CPU features specific to the target platform and exit
* `--list-modules`: list observable modules and exit
* `--native-compiler-options`: provide a custom C compiler option used for query code compilation
* `--native-compiler-path`: provide a custom path to the C compiler used to query code compilation and linking
* `--native-image-info`: show the native toolchain information and executable's build settings
* `--no-fallback`: build a standalone native executable or report a failure
* `--pgo`: a comma-separated list of files from which to read the data collected for profile-guided optimizations of AOT-compiled code (reads from  _default.iprof_ if nothing is specified). (Not available in GraalVM Community Edition.)
* `--pgo-instrument`: instrument AOT-compiled code to collect data for profile-guided optimizations into the _default.iprof_ file. (Not available in GraalVM Community Edition.)
* `--report-unsupported-elements-at-runtime`: report the usage of unsupported methods and fields at run time when they are accessed the first time, instead of an error during executable's building
* `--shared`: build a shared library
* `--silent`: silence build output
* `--static`: build a statically-linked executable (requires `libc` and `zlib` static libraries)
* `--target`: select the compilation target for `native-image` (in <OS>-<architecture> format). It defaults to host's OS-architecture pair.
* `--trace-class-initialization`: provide a comma-separated list of fully-qualified class names that a class initialization is traced for
* `--trace-object-instantiation`: provide a comma-separated list of fully-qualified class names that an object instantiation is traced for
* `-O<level>`: control code optimizations where available variants are `b` for quick build mode for development, `0` - no optimizations, `1` - basic optimizations, `2` - aggressive optimizations (default)
* `-da`, `-da[:[packagename]|:[classname]`, `disableassertions[:[packagename]|:[classname]`: disable assertions with specified granularity at run time
* `-dsa`, `-disablesystemassertions`: disable assertions in all system classes at run time
* `-ea`, `-ea[:[packagename]|:[classname]`, `enableassertions[:[packagename]|:[classname]`: enable assertions with specified granularity at run time
* `-esa`, `-enablesystemassertions`: enable assertions in all system classes at run time
* `-g`: generate debugging information
* `-march`: generate instructions for a specific machine type. Defaults to `x86-64-v3` on AMD64 and `armv8-a` on AArch64. Use `-march=compatibility` for best compatibility, or `-march=native` for best performance if a native executable is deployed on the same machine or on a machine with the same CPU features. To list all available machine types, use `-march=list`.
* `-o`: name of the output file to be generated

### Macro Options

* `--language:nfi`: make the Truffle Native Function Interface language available
* `--tool:coverage`: add source code coverage support to a GraalVM-supported language
* `--tool:insight`: add support for detailed access to program's runtime behavior, allowing users to inspect values and types at invocation or allocation sites
* `--tool:dap`: allow image to open a debugger port serving the Debug Adapter Protocol in IDEs like Visual Studio Code
* `--tool:chromeinspector`: add debugging support to a GraalVM-supported language
* `--tool:insightheap`: snapshot a region of image heap during the execution
* `--tool:lsp`: add the Language Server Protocol support to later attach compatible debuggers to GraalVM in IDEs like Visual Studio Code
* `--tool:sandbox`: enables the Truffle sandbox resource limits. For more information, check the [dedicated documentation](../../security/polyglot-sandbox.md)
* `--tool:profiler`: add profiling support to a GraalVM-supported language

The `--language:js` `--language:nodejs`, `--language:python`, `--language:ruby`, `--language:R`, `--language:wasm`, `--language:llvm`, `--language:regex` (enables the Truffle Regular Expression engine) polyglot macro options become available once the corresponding languages are added to the base GraalVM JDK.

### Non-standard Options

Run `native-image --help-extra` for non-standard options help.

* `--expert-options`: list image build options for experts
* `--expert-options-all `: list all image build options for experts (use at your own risk). Options marked with _Extra help available_ contain help that can be shown with `--expert-options-detail`
* `--expert-options-detail`: display all available help for a comma-separated list of option names. Pass `*` to show extra help for all options that contain it.
* `--configurations-path <search path of option-configuration directories>`: a separated list of directories to be treated as option-configuration directories.
* `--debug-attach[=< port (* can be used as host meaning bind to all interfaces)>]`: attach to debugger during image building (default port is 8000)
* `--diagnostics-mode`: Enables logging of image-build information to a diagnostics folder.
* `--dry-run`: output the command line that would be used for building
* `--bundle-create[=new-bundle.nib]`: in addition to image building, create a native image bundle file _(*.nibfile)_ that allows rebuilding of that image again at a later point. If a bundle-file gets passed the bundle will be created with the given name. Otherwise, the bundle-file name is derived from the image name. Note both bundle options can be combined with `--dry-run` to only perform the bundle operations without any actual image building.
* `--bundle-apply=some-bundle.nib`: an image will be built from the given bundle file with the exact same arguments and files that have been passed to Native Image originally to create the bundle. Note that if an extra `--bundle-create` gets passed after `--bundle-apply`, a new bundle will be written based on the given bundle args plus any additional arguments that haven been passed afterwards. For example: `native-image --bundle-apply=app.nib --bundle-create=app_dbg.nib -g` creates a new bundle <app_dbg.nib>  based on the given _app.nib_ bundle. Both bundles are the same except the new one also uses the -g option.
* `-E<env-var-key>[=<env-var-value>]`: allow Native Image to access the given environment variable during image build. If the optional <env-var-value> is not given, the value of the environment variable will be taken from the environment Native Image was invoked from.
* `-V<key>=<value>`:  provide values for placeholders in `native-image.properties` files
* `--add-exports`: value `<module>/<package>=<target-module>(,<target-module>)` updates `<module>` to export `<package>` to `<target-module>`, regardless of module declaration. `<target-module>` can be `ALL-UNNAMED` to export to all unnamed modules
* `--add-opens`: value `<module>/<package>=<target-module>(,<target-module>)` updates `<module>` to open `<package>` to `<target-module>`, regardless of module declaration. 
* `--add-reads`: value `<module>=<target-module>(,<target-module>)` updates `<module>` to read `<target-module>`, regardless of module declaration. `<target-module>` can be `ALL-UNNAMED` to read all unnamed modules.

Native Image options are also distinguished as [hosted and runtime options](HostedvsRuntimeOptions.md).

# Native Image Build Output

* [Build Stages](#build-stages)
* [Resource Usage Statistics](#resource-usage-statistics)
* [Machine-Readable Build Output](#machine-readable-build-output)

Here you will find information about the build output of GraalVM Native Image.
Below is the example output when building a native executable of the `HelloWorld` class:

```shell
================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
================================================================================
[1/8] Initializing...                                            (2.8s @ 0.15GB)
 Java version: 20+34, vendor version: GraalVM CE 20-dev+34.1
 Graal compiler: optimization level: 2, target machine: x86-64-v3
 C compiler: gcc (linux, x86_64, 12.2.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
--------------------------------------------------------------------------------
 Build resources:
 - 13.24GB of memory (42.7% of 31.00GB system memory, determined at start)
 - 16 thread(s) (100.0% of 16 available processor(s), determined at start)
[2/8] Performing analysis...  [****]                             (4.5s @ 0.54GB)
    3,163 reachable types   (72.5% of    4,364 total)
    3,801 reachable fields  (50.3% of    7,553 total)
   15,183 reachable methods (45.5% of   33,405 total)
      957 types,    81 fields, and   480 methods registered for reflection
       57 types,    55 fields, and    52 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
[3/8] Building universe...                                       (0.8s @ 0.99GB)
[4/8] Parsing methods...      [*]                                (0.6s @ 0.75GB)
[5/8] Inlining methods...     [***]                              (0.3s @ 0.32GB)
[6/8] Compiling methods...    [**]                               (3.7s @ 0.60GB)
[7/8] Layouting methods...    [*]                                (0.8s @ 0.83GB)
[8/8] Creating image...       [**]                               (3.1s @ 0.58GB)
   5.32MB (24.22%) for code area:     8,702 compilation units
   7.03MB (32.02%) for image heap:   93,301 objects and 5 resources
   8.96MB (40.83%) for debug info generated in 1.0s
 659.13kB ( 2.93%) for other data
  21.96MB in total
--------------------------------------------------------------------------------
Top 10 origins of code area:            Top 10 object types in image heap:
   4.03MB java.base                        1.14MB byte[] for code metadata
 927.05kB svm.jar (Native Image)         927.31kB java.lang.String
 111.71kB java.logging                   839.68kB byte[] for general heap data
  63.38kB org.graalvm.nativeimage.base   736.91kB java.lang.Class
  47.59kB jdk.proxy1                     713.13kB byte[] for java.lang.String
  35.85kB jdk.proxy3                     272.85kB c.o.s.c.h.DynamicHubCompanion
  27.06kB jdk.internal.vm.ci             250.83kB java.util.HashMap$Node
  23.44kB org.graalvm.sdk                196.52kB java.lang.Object[]
  11.42kB jdk.proxy2                     182.77kB java.lang.String[]
   8.07kB jdk.internal.vm.compiler       154.26kB byte[] for embedded resources
   1.39kB for 2 more packages              1.38MB for 884 more object types
--------------------------------------------------------------------------------
Recommendations:
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
--------------------------------------------------------------------------------
    0.8s (4.6% of total time) in 35 GCs | Peak RSS: 1.93GB | CPU load: 9.61
--------------------------------------------------------------------------------
Produced artifacts:
 /home/janedoe/helloworld/helloworld (executable)
 /home/janedoe/helloworld/helloworld.debug (debug_info)
 /home/janedoe/helloworld/sources (debug_info)
================================================================================
Finished generating 'helloworld' in 17.0s.
```

## Build Stages

### <a name="stage-initializing"></a>Initializing
In this stage, the Native Image build process is set up and [`Features`](https://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/hosted/Feature.html) are initialized.

#### <a name="glossary-imagekind"></a>Native Image Kind
By default, Native Image generates *executables* but it can also generate [*native shared libraries*](InteropWithNativeCode.md) and [*static executables*](guides/build-static-and-mostly-static-executable.md).

#### <a name="glossary-java-info"></a>Java Version Info
The Java and vendor version of the Native Image process.
Both are also used for the `java.vm.version` and `java.vendor.version` properties within the generated native binary.
Please report version and vendor when you [file issues](https://github.com/oracle/graal/issues/new).

#### <a name="glossary-graal-compiler"></a>Graal Compiler
The selected optimization level and targeted machine type used by the Graal compiler.
The optimization level can be controlled with the `-O` option and defaults to `2`, which enables aggressive optimizations.
Use `-Ob` to enable quick build mode, which speeds up the [compilation stage](#stage-compiling).
This is useful during development, or when peak throughput is less important and you would like to optimize for size.
The targeted machine type can be selected with the `-march` option and defaults to `x86-64-v3` on AMD64 and `armv8-a` on AArch64.
See [here](#recommendation-cpu) for recommendations on how to use this option.

On Oracle GraalVM, the line also shows information about [Profile-Guided Optimizations (PGO)](#recommendation-pgo).
- `off`: PGO is not used
- `instrument`: The generated executable or shared library is instrumented to collect data for PGO (`--pgo-instrument`)
- `user-provided`: PGO is enabled and uses a user-provided profile (for example `--pgo default.iprof`)
- `ML-inferred`: A machine learning (ML) model is used to infer profiles for control split branches statically.

#### <a name="glossary-ccompiler"></a>C Compiler
The C compiler executable, vendor, target architecture, and version info used by the Native Image build process.

#### <a name="glossary-gc"></a>Garbage Collector
The garbage collector used within the generated executable:
- The *Serial GC* is the default GC and optimized for low memory footprint and small Java heap sizes.
- The *G1 GC* (not available in GraalVM Community Edition) is a multi-threaded GC that is optimized to reduce stop-the-world pauses and therefore improve latency while achieving high throughput.
- The *Epsilon GC* does not perform any garbage collection and is designed for very short-running applications that only allocate a small amount of memory.

For more information see the [docs on Memory Management](MemoryManagement.md).

#### <a name="glossary-gc-max-heap-size"></a>Maximum Heap Size
By default, the heap size is limited to a certain percentage of your system memory, allowing the garbage collector to freely allocate memory according to its policy.
Use the `-Xmx` option when invoking your native executable (for example `./myapp -Xmx64m` for 64MB) to limit the maximum heap size for a lower and more predictable memory footprint.
This can also improve latency in some cases.
Use the `-R:MaxHeapSize` option when building with Native Image to pre-configure the maximum heap size.

#### <a name="glossary-user-specific-features"></a>User-Specific Features
All [`Features`](https://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/hosted/Feature.html) that are either provided or specifically enabled by the user, or implicitly registered for the user, for example, by a framework.
GraalVM Native Image deploys a number of internal features, which are excluded from this list.

#### <a name="glossary-build-resources"></a>Build Resources
The memory limit and number of threads used by the build process.

More precisely, the memory limit of the Java heap, so actual memory consumption can be even higher.
Please check the [peak RSS](#glossary-peak-rss) reported at the end of the build to understand how much memory was actually used.
By default, the process will only use _available_ memory: memory that the operating system can make available without swapping out memory used by other processes.
Therefore, consider freeing up memory if your build process is slow, for example, by closing applications that you do not need.
Note that, by default, the build process will not use more than 32GB available memory.

By default, the build process uses all available CPU cores to maximize speed.
Use the `--parallelism` option to set the number of threads explicitly (for example, `--parallelism=4`).
Use fewer threads to reduce load on your system as well as memory consumption (at the cost of a slower build process).

### <a name="stage-analysis"></a>Performing Analysis
In this stage, a [points-to analysis](https://dl.acm.org/doi/10.1145/3360610) is performed.
The progress indicator visualizes the number of analysis iterations.
A large number of iterations can indicate problems in the analysis likely caused by misconfiguration or a misbehaving feature.

#### <a name="glossary-reachability"></a>Reachable Types, Fields, and Methods
The number of types (primitives, classes, interfaces, and arrays), fields, and methods that are reachable versus the total number of types, fields, and methods loaded as part of the build process.
A significantly larger number of loaded elements that are not reachable can indicate a configuration problem.
To reduce overhead, please ensure that your class path and module path only contain entries that are needed for building the application.

#### <a name="glossary-reflection-registrations"></a>Reflection Registrations
The number of types, fields, and methods that are registered for reflection.
Large numbers can cause significant reflection overheads, slow down the build process, and increase the size of the native binary (see [reflection metadata](#glossary-reflection-metadata)).

#### <a name="glossary-jni-access-registrations"></a>JNI Access Registrations
The number of types, fields, and methods that are registered for [JNI](JNI.md) access.

#### <a name="glossary-runtime-methods"></a>Runtime Compiled Methods
The number of methods marked for runtime compilation.
This number is only shown if runtime compilation is built into the executable, for example, when building a [Truffle](https://github.com/oracle/graal/tree/master/truffle) language.
Runtime-compiled methods account for [graph encodings](#glossary-graph-encodings) in the heap.

### <a name="stage-universe"></a>Building Universe
In this stage, a universe with all types, fields, and methods is built, which is then used to create the native binary.

### <a name="stage-parsing"></a>Parsing Methods
In this stage, the Graal compiler parses all reachable methods.
The progress indicator is printed periodically at an increasing interval.

### <a name="stage-inlining"></a>Inlining Methods
In this stage, trivial method inlining is performed.
The progress indicator visualizes the number of inlining iterations.

### <a name="stage-compiling"></a>Compiling Methods
In this stage, the Graal compiler compiles all reachable methods to machine code.
The progress indicator is printed periodically at an increasing interval.

### <a name="stage-layouting"></a>Layouting Methods
In this stage, compiled methods are layouted.
The progress indicator is printed periodically at an increasing interval.

### <a name="stage-creating"></a>Creating Image
In this stage, the native binary is created and written to disk.
Debug info is also generated as part of this stage (if requested).

#### <a name="glossary-code-area"></a>Code Area
The code area contains machine code produced by the Graal compiler for all reachable methods.
Therefore, reducing the number of [reachable methods](#glossary-reachability) also reduces the size of the code area.

##### <a name="glossary-code-area-origins"></a>Origins of Code Area
To help users understand where the machine code of the code area comes from, the build output shows a breakdown of the top origins.
An origin is a group of Java sources and can be a JAR file, a package name, or a class name, depending on the information available.
The [`java.base` module](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/module-summary.html), for example, contains base classes from the JDK.
The `svm.jar` file, the `org.graalvm.nativeimage.base` module, and similar origins contain internal sources for the Native Image runtime.
To reduce the size of the code area and with that, the total size of the native executable, re-evaluate the dependencies of your application based on the code area breakdown.
Some libraries and frameworks are better prepared for Native Image than others, and newer versions of a library or framework may improve (or worsen) their code footprint. 

#### <a name="glossary-image-heap"></a>Image Heap
The heap contains reachable objects such as static application data, metadata, and `byte[]` for different purposes (see below).

##### <a name="glossary-general-heap-data"></a>General Heap Data Stored in `byte[]`
The total size of all `byte[]` objects that are neither used for `java.lang.String`, nor [code metadata](#glossary-code-metadata), nor [reflection metadata](#glossary-reflection-metadata), nor [graph encodings](#glossary-graph-encodings).
Therefore, this can also include `byte[]` objects from application code.

##### <a name="glossary-embedded-resources"></a>Embedded Resources Stored in `byte[]`
The total size of all `byte[]` objects used for storing resources (for example, files accessed via `Class.getResource()`) within the native binary. The number of resources is shown in the [Heap](#glossary-image-heap) section.

##### <a name="glossary-code-metadata"></a>Code Metadata Stored in `byte[]`
The total size of all `byte[]` objects used for metadata for the [code area](#glossary-code-area).
Therefore, reducing the number of [reachable methods](#glossary-reachability) also reduces the size of this metadata.

##### <a name="glossary-reflection-metadata"></a>Reflection Metadata Stored in `byte[]`
The total size of all `byte[]` objects used for reflection metadata, including types, field, method, and constructor data.
To reduce the amount of reflection metadata, reduce the number of [elements registered for reflection](#glossary-reflection-registrations).

##### <a name="glossary-graph-encodings"></a>Graph Encodings Stored in `byte[]`
The total size of all `byte[]` objects used for graph encodings.
These encodings are a result of [runtime compiled methods](#glossary-runtime-methods).
Therefore, reducing the number of such methods also reduces the size of corresponding graph encodings.

#### <a name="glossary-debug-info"></a>Debug Info
The total size of generated debug information (if enabled).

#### <a name="glossary-other-data"></a>Other Data
The amount of data in the binary that is neither in the [code area](#glossary-code-area), nor in the [heap](#glossary-image-heap), nor [debug info](#glossary-debug-info).
This data typically contains internal information for Native Image and should not be dominating.

## Recommendations

The build output may contain one or more of the following recommendations that help you get the best out of Native Image.

#### <a name="recommendation-awt"></a>`AWT`: Missing Reachability Metadata for Abstract Window Toolkit

The Native Image analysis has included classes from the [`java.awt` package](https://docs.oracle.com/en/java/javase/17/docs/api/java.desktop/java/awt/package-summary.html) but could not find any reachability metadata for it.
Use the [tracing agent](AutomaticMetadataCollection.md) to collect such metadata for your application.
Otherwise, your application is unlikely to work properly.
If your application is not a desktop application (for example using Swing or AWT directly), you may want to re-evaluate whether the dependency on AWT is actually needed.

#### <a name="recommendation-cpu"></a>`CPU`: Enable More CPU Features for Improved Performance

The Native Image build process has determined that your CPU supports more features, such as AES or LSE, than currently enabled.
If you deploy your application on the same machine or a similar machine with support for the same CPU features, consider using `-march=native` at build time.
This option allows the Graal compiler to use all CPU features available, which in turn can significantly improve the performance of your application.
Use `-march=list` to list all available machine types that can be targeted explicitly.

#### <a name="recommendation-g1gc"></a>`G1GC`: Use G1 Garbage Collector for Improved Latency and Throughput

The G1 garbage collector is available for your platform.
Consider enabling it using `--gc=G1` at build time to improve the latency and throughput of your application.
For more information see the [docs on Memory Management](MemoryManagement.md).
For best peak performance, also consider using [Profile-Guided Optimizations](#recommendation-pgo).

#### <a name="recommendation-heap"></a>`HEAP`: Specify a Maximum Heap Size

Please refer to [Maximum Heap Size](#glossary-gc-max-heap-size).

#### <a name="recommendation-pgo"></a>`PGO`: Use Profile-Guided Optimizations for Improved Throughput

Consider using Profile-Guided Optimizations to optimize your application for improved throughput.
These optimizations allow the Graal compiler to leverage profiling information, similar to when it is running as a JIT compiler, when AOT-compiling your application.
For this, perform the following steps:

1. Build your application with `--pgo-instrument`.
2. Run your instrumented application with a representative workload to generate profiling information in the form of an `.iprof` file.
3. Re-build your application and pass in the profiling information with `--pgo=<your>.iprof` to generate an optimized version of your application.

Relevant guide: [Optimize a Native Executable with Profile-Guided Optimizations](guides/optimize-native-executable-with-pgo.md).

For best peak performance, also consider using the [G1 garbage collector](#recommendation-g1gc).

#### <a name="recommendation-qbm"></a>`QBM`: Use Quick Build Mode for Faster Builds

Consider using the quick build mode (`-Ob`) to speed up your builds during development.
More precisely, this mode reduces the number of optimizations performed by the Graal compiler and thus reduces the overall time of the [compilation stage](#stage-compiling).
The quick build mode is not only useful for development, it can also cause the generated executable file to be smaller in size.
Note, however, that the overall peak throughput of the executable may be lower due to the reduced number of optimizations.


## Resource Usage Statistics

#### <a name="glossary-garbage-collection"></a>Garbage Collections
The total time spent in all garbage collectors, total GC time divided by the total process time as a percentage, and the total number of garbage collections.
A large number of collections or time spent in collectors usually indicates that the system is under memory pressure.
Increase the amount of available memory to reduce the time to build the native binary.

#### <a name="glossary-peak-rss"></a>Peak RSS
Peak [resident set size](https://en.wikipedia.org/wiki/Resident_set_size) as reported by the operating system.
This value indicates the maximum amount of memory consumed by the build process.
You may want to compare this value to the memory limit reported in the [build resources section](#glossary-build-resources).
If there is enough headroom and the [GC statistics](#glossary-garbage-collection) do not show any problems, the amount of total memory of the system can be reduced to a value closer to the peak RSS to lower operational costs.

#### <a name="glossary-cpu-load"></a>CPU load
The CPU time used by the process divided by the total process time.
Increase the number of CPU cores to reduce the time to build the native binary.

## Machine-Readable Build Output

The build output produced by the `native-image` builder is designed for humans, can evolve with new releases, and should thus not be parsed in any way by tools.
Instead, use the `-H:BuildOutputJSONFile=<file.json>` option to instruct the builder to produce machine-readable build output in JSON format that can be used, for example, for building monitoring tools.
The JSON files validate against the JSON schema defined in [`build-output-schema-v0.9.2.json`](https://github.com/oracle/graal/tree/master/docs/reference-manual/native-image/assets/build-output-schema-v0.9.2.json).
Note that a JSON file is produced if and only if a build succeeds.

The following example illustrates how this could be used in a CI/CD build pipeline to check that the number of reachable methods does not exceed a certain threshold:

```bash
native-image -H:BuildOutputJSONFile=build.json HelloWorld
# ...
cat build.json | python3 -c "import json,sys;c = json.load(sys.stdin)['analysis_results']['methods']['reachable']; assert c < 12000, f'Too many reachable methods: {c}'"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
AssertionError: Too many reachable methods: 12128
```

## Related Documentation

- [Build a Native Shared Library](guides/build-native-shared-library.md)
- [Build a Statically Linked or Mostly-Statically Linked Native Executable](guides/build-static-and-mostly-static-executable.md)
- [Feature](https://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/hosted/Feature.html)
- [Interoperability with Native Code](InteropWithNativeCode.md)
- [Java Native Interface (JNI) in Native Image](JNI.md)
- [Memory Management](MemoryManagement.md)
- [Native Image Build Overview](BuildOverview.md)
- [Native Image Build Configuration](BuildConfiguration.md)

# Native Image Build Overview

The syntax of the `native-image` command is:

- `native-image [options] <mainclass> [imagename] [options]` to build a native binary from `<mainclass>` class in the current working directory. The classpath may optionally be provided with the `-cp <classpath>` option where `<classpath>` is a colon-separated (on Windows, semicolon-separated) list of paths to directories and jars.
- `native-image [options] -jar jarfile [imagename] [options]` to build native binary from a JAR file.
- `native-image [options] -m <module>/<mainClass> [imagename] [options]` to build a native binary from a Java module.

The options passed to `native-image` are evaluated from left to right.

The options fall into three categories:
 - [Image generation options](BuildOptions.md#native-image-build-options) - for the full list, run `native-image --help`
 - [Macro options](BuildOptions.md#macro-options)
 - [Non-standard options](BuildOptions.md#non-standard-options) - subject to change through a deprecation cycle, run `native-image --help-extra` for the full list.

Find a complete list of options for the `native-image` tool [here](BuildOptions.md).

There are some expert level options that a Native Image developer may find useful or needed, for example, the option to dump graphs of the `native-image` builder or enable assertions at image run time. This information can be found in [Native Image Hosted and Runtime Options](HostedvsRuntimeOptions.md).

### Further Reading

If you are new to GraalVM Native Image or have little experience using it, see the [Native Image Basics](NativeImageBasics.md) to better understand some key aspects before going further.

For more tweaks and how to properly configure the `native-image` tool, see [Build Configuration](BuildConfiguration.md#order-of-arguments-evaluation).

Native Image will output the progress and various statistics when building the native binary. To learn more about the output, and the different build phases, see [Build Output](BuildOutput.md).

# Native Image Bundles

Native Image provides a feature that enables users to build native executables from a self-contained _bundle_. 
In contrast to regular `native-image` building, this mode of operation takes only a single _*.nib_ file as input.
The file contains everything required to build a native executable (or a native shared library).
This can be useful when large applications consisting of many input files (JAR files, configuration files, auto-generated files, downloaded files) need to be rebuilt at a later point in time without worrying whether all files are still available.
Often complex builds involve downloading many libraries that are not guaranteed to remain accessible later in time.
Using Native Image bundles is a safe solution to encapsulate all this input required for building into a single file.

> Note: The feature is experimental.

### Table of Contents

* [Creating Bundles](#creating-bundles)
* [Building with Bundles](#building-with-bundles)
* [Environment Variables](#capturing-environment-variables)
* [Creating New Bundles from Existing Bundles](#combining---bundle-create-and---bundle-apply)
* [Bundle File Format](#bundle-file-format)

## Creating Bundles

To create a bundle, pass the `--bundle-create` option along with the other arguments for a specific `native-image` command line invocation.
This will cause `native-image` to create a _*.nib_ file in addition to the actual image.

Here is the option description:
```
--bundle-create[=new-bundle.nib]
                      in addition to image building, create a Native Image bundle file (*.nib
                      file) that allows rebuilding of that image again at a later point. If a
                      bundle-file gets passed, the bundle will be created with the given
                      name. Otherwise, the bundle-file name is derived from the image name.
                      Note both bundle options can be combined with --dry-run to only perform
                      the bundle operations without any actual image building.
```

For example, assuming a Micronaut application is built with Maven, make sure the `--bundle-create` option is used.
For that, the following needs to be added to the plugins section of `pom.xml`:
```xml
<plugin>
  <groupId>org.graalvm.buildtools</groupId>
  <artifactId>native-maven-plugin</artifactId>
  <configuration>
      <buildArgs combine.children="append">
          <buildArg>--bundle-create</buildArg>
      </buildArgs>
  </configuration>
</plugin>
```

Then, when you run the Maven package command `./mvnw package -Dpackaging=native-image`, you will get the following build artifacts:
```
Finished generating 'micronautguide' in 2m 0s.

Native Image Bundles: Bundle build output written to /home/testuser/micronaut-data-jdbc-repository-maven-java/target/micronautguide.output
Native Image Bundles: Bundle written to /home/testuser/micronaut-data-jdbc-repository-maven-java/target/micronautguide.nib

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  02:08 min
[INFO] Finished at: 2023-03-27T15:09:36+02:00
[INFO] ------------------------------------------------------------------------
```

This output indicates that you have created a native executable, `micronautguide`, and a bundle, _micronautguide.nib_.
The bundle file is created in the _target/_ directory.
It should be copied to some safe place where it can be found if the native executable needs to be rebuilt later.

Obviously, a bundle file can be large because it contains all input files as well as the executable itself (the executable is compressed within the bundle). 
Having the image inside the bundle allows comparing a native executable rebuilt from the bundle against the original one.
In the case of the `micronaut-data-jdbc-repository` example, the bundle is 60.7 MB (the executable is 103.4 MB).
To see what is inside a bundle, run `jar tf *.nib`:
```shell
$ jar tf micronautguide.nib
META-INF/MANIFEST.MF
META-INF/nibundle.properties
output/default/micronautguide
input/classes/cp/micronaut-core-3.8.7.jar
input/classes/cp/netty-buffer-4.1.87.Final.jar
input/classes/cp/jackson-databind-2.14.1.jar
input/classes/cp/micronaut-context-3.8.7.jar
input/classes/cp/reactive-streams-1.0.4.jar
...
input/classes/cp/netty-handler-4.1.87.Final.jar
input/classes/cp/micronaut-jdbc-4.7.2.jar
input/classes/cp/jackson-core-2.14.0.jar
input/classes/cp/micronaut-runtime-3.8.7.jar
input/classes/cp/micronautguide-0.1.jar
input/stage/build.json
input/stage/environment.json
input/stage/path_substitutions.json
input/stage/path_canonicalizations.json
```

As you can see, a bundle is just a JAR file with a specific layout.
This is explained in detail [below](#bundle-file-format).

Next to the bundle, you can also find the output directory: _target/micronautguide.output_.
It contains the native executable and all other files that were created as part of the build. 
Since you did not specify any options that would produce extra output (for example, `-g` to generate debugging information or `--diagnostics-mode`), only the executable can be found there:
```shell
$ tree target/micronautguide.output
target/micronautguide.output
├── default
│   └── micronautguide
└── other
```

### Combining --bundle-create with --dry-run

As mentioned in the `--bundle-create` option description, it is also possible to let `native-image` build a bundle but not actually perform the image building.
This might be useful if a user wants to move the bundle to a more powerful machine and build the image there.
Modify the above `native-maven-plugin` configuration to also contain the argument `<buildArg>--dry-run</buildArg>`. 
Then running `./mvnw package -Dpackaging=native-image` takes only seconds and the created bundle is much smaller: 
```
Native Image Bundles: Bundle written to /home/testuser/micronaut-data-jdbc-repository-maven-java/target/micronautguide.nib

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.267 s
[INFO] Finished at: 2023-03-27T16:33:21+02:00
[INFO] ------------------------------------------------------------------------
```

Now _micronautguide.nib_ is only 20 MB in file size and the executable is not included:
```shell
$ jar tf micronautguide.nib
META-INF/MANIFEST.MF
META-INF/nibundle.properties
input/classes/cp/micronaut-core-3.8.7.jar
...
```

Note that this time you do not see the following message in the Maven output:
```
Native Image Bundles: Bundle build output written to /home/testuser/micronaut-data-jdbc-repository-maven-java/target/micronautguide.output
```
Since no executable is created, no bundle build output is available.

## Building with Bundles

Assuming that the native executable is used in production and once in a while an unexpected exception is thrown at run time.
Since you still have the bundle that was used to create the executable, it is trivial to build a variant of that executable with debugging support.
Use `--bundle-apply=micronautguide.nib` like this:
```shell
$ native-image --bundle-apply=micronautguide.nib -g

Native Image Bundles: Loaded Bundle from /home/testuser/micronautguide.nib
Native Image Bundles: Bundle created at 'Tuesday, March 28, 2023, 11:12:04 AM Central European Summer Time'
Native Image Bundles: Using version: '20.0.1+8' (vendor 'Oracle Corporation') on platform: 'linux-amd64'
Warning: Native Image Bundles are an experimental feature.
========================================================================================================================
GraalVM Native Image: Generating 'micronautguide' (executable)...
========================================================================================================================
...
Finished generating 'micronautguide' in 2m 16s.

Native Image Bundles: Bundle build output written to /home/testuser/micronautguide.output
```

After running this command, the executable is rebuilt with an extra option `-g` passed after `--bundle-apply`.
The output of this build is in the directory _micronautguide.output_:
```
micronautguide.output
micronautguide.output/other
micronautguide.output/default
micronautguide.output/default/micronautguide.debug
micronautguide.output/default/micronautguide
micronautguide.output/default/sources
micronautguide.output/default/sources/javax
micronautguide.output/default/sources/javax/smartcardio
micronautguide.output/default/sources/javax/smartcardio/TerminalFactory.java
...
micronautguide.output/default/sources/java/lang/Object.java
```

You successfully rebuilt the application from the bundle with debug info enabled.

The full option help of `--bundle-apply` shows a more advanced use case that will be discussed [later](#combining---bundle-create-and---bundle-apply) in detail:
```
--bundle-apply=some-bundle.nib
                      an image will be built from the given bundle file with the exact same
                      arguments and files that have been passed to native-image originally
                      to create the bundle. Note that if an extra --bundle-create gets passed
                      after --bundle-apply, a new bundle will be written based on the given
                      bundle args plus any additional arguments that haven been passed
                      afterwards. For example:
                      > native-image --bundle-apply=app.nib --bundle-create=app_dbg.nib -g
                      creates a new bundle app_dbg.nib based on the given app.nib bundle.
                      Both bundles are the same except the new one also uses the -g option.
```

## Capturing Environment Variables

Before bundle support was added, all environment variables were visible to the  `native-image` builder.
This approach does not work well with bundles and is problematic for image building without bundles.
Consider having an environment variable that holds sensitive information from your build machine.
Due to Native Image's ability to run code at build time that can create data to be available at run time, it is very easy to build an image were you accidentally leak the contents of such variables.

Passing environment variables to `native-image` now requires explicit arguments.

Suppose a user wants to use an environment variable (for example, `KEY_STORAGE_PATH`) from the environment in which the `native-image` tool is invoked, in the class initializer that is set to be initialized at build time.
To allow accessing the variable in the class initializer (with `java.lang.System.getenv`), pass the option `-EKEY_STORAGE_PATH` to the builder.

To make an environment variable accessible to build time, use:
```
-E<env-var-key>[=<env-var-value>]
                      allow native-image to access the given environment variable during
                      image build. If the optional <env-var-value> is not given, the value
                      of the environment variable will be taken from the environment
                      native-image was invoked from.
```

Using `-E` works as expected with bundles.
Any environment variable specified with `-E` will be captured in the bundle.
For variables where the optional `<env-var-value>` is not given, the bundle would capture the value the variable had at the time the bundle was created.
The prefix `-E` was chosen to make the option look similar to the related `-D<java-system-property-key>=<java-system-property-value>` option (which makes Java system properties available at build time).

## Combining --bundle-create and --bundle-apply

As already mentioned in [Building with Bundles](#building-with-bundles), it is possible to create a new bundle based on an existing one.
The `--bundle-apply` help message has a simple example.
A more interesting example arises if an existing bundle is used to create a new bundle that builds a PGO-optimized version of the original application.

Assuming you have already built the `micronaut-data-jdbc-repository` example into a bundle named _micronautguide.nib_.
To produce a PGO-optimized variant of that bundle, first build a variant of the native executable that generates PGO profiling information at run time (you will use it later):
```shell
$ native-image --bundle-apply=micronautguide.nib --pgo-instrument
```

Now run the generated executable so that profile information is collected:
```shell
$ /home/testuser/micronautguide.output/default/micronautguide
```

Based on <a href="https://guides.micronaut.io/latest/micronaut-data-jdbc-repository.html" target="_blank">this walkthrough</a>, you use the running native executable to add new database entries and query the information in the database afterwards so that you get real-world profiling information.
Once completed, stop the Micronaut application using `Ctrl+C` (`SIGTERM`).
Looking into the current working directory, you can find a new file:
```shell
$ ls -lh  *.iprof
-rw------- 1 testuser testuser 19M Mar 28 14:52 default.iprof
```

The file `default.iprof` contains the profiling information that was created because you ran the Micronaut application from the executable built with `--pgo-instrument`.
Now you can create a new optimized bundle out of the existing one:
```shell
native-image --bundle-apply=micronautguide.nib --bundle-create=micronautguide-pgo-optimized.nib --dry-run --pgo
```

Now take a look how _micronautguide-pgo-optimized.nib_ is different from _micronautguide.nib_:
```shell
$ ls -lh *.nib
-rw-r--r-- 1 testuser testuser  20M Mar 28 11:12 micronautguide.nib
-rw-r--r-- 1 testuser testuser  23M Mar 28 15:02 micronautguide-pgo-optimized.nib
```

You can see that the new bundle is 3 MB larger than the original.
The reason, as can be guessed, is that now the bundle contains the _default.iprof_ file.
Using a tool to compare directories, you can inspect the differences in detail:

![visual-bundle-compare](visual-bundle-compare.png)

As you can see, _micronautguide-pgo-optimized.nib_ contains _default.iprof_ in the directory _input/auxiliary_, and there
are also changes in other files. The contents of _META-INF/nibundle.properties_, _input/stage/path_substitutions.json_
and _input/stage/path_canonicalizations.json_ will be explained [later](#bundle-file-format). 
For now, look at the diff in _build.json_:
```
@@ -4,5 +4,6 @@
   "--no-fallback",
   "-H:Name=micronautguide",
   "-H:Class=example.micronaut.Application",
-  "--no-fallback"
+  "--no-fallback",
+  "--pgo"
 ]
```

As expected, the new bundle contains the `--pgo` option that you passed to `native-image` to build an optimized bundle.
Building a native executable from this new bundle generates a PGO-optimized executable out of the box (see `PGO: on` in build output):
```shell
$ native-image --bundle-apply=micronautguide-pgo-optimized.nib
...
[1/8] Initializing...                                                                                    (3.9s @ 0.27GB)
 Java version: 20.0.1+8, vendor version: GraalVM EE 20.0.1+8.1
 Graal compiler: optimization level: '2', target machine: 'x86-64-v3', PGO: on
 C compiler: gcc (redhat, x86_64, 13.0.1)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
 6 user-specific feature(s)
...
```

## Bundle File Format

A bundle file is a JAR file with a well-defined internal layout.
Inside a bundle you can find the following inner structure:

```
[bundle-file.nib]
├── META-INF
│   ├── MANIFEST.MF
│   └── nibundle.properties <- Contains build bundle version info:
│                              * Bundle format version (BundleFileVersion{Major,Minor})
│                              * Platform and architecture the bundle was created on 
│                              * GraalVM / Native-image version used for bundle creation
├── input <- All information required to rebuild the image
│   ├── auxiliary <- Contains auxiliary files passed to native-image via arguments
│   │                (e.g. external `config-*.json` files or PGO `*.iprof`-files)
│   ├── classes   <- Contains all class-path and module-path entries passed to the builder
│   │   ├── cp
│   │   └── p
│   └── stage
│       ├── build.json          <- Full native-image command line (minus --bundle options)
│       ├── environment.json              <- Environment variables used in the image build
│       ├── path_canonicalizations.json  <- Record of path-canonicalizations that happened
│       │                                       during bundle creation for the input files  
│       └── path_substitutions.json          <- Record of path-substitutions that happened
│                                               during bundle creation for the input files
└── output
    ├── default
    │   ├── myimage         <- Created image and other output created by the image builder 
    │   ├── myimage.debug
    |   └── sources
    └── other      <- Other output created by the builder (not relative to image location)
```
### META-INF

The layout of a bundle file itself is versioned.
There are two properties in _META-INF/nibundle.properties_ that declare which version of the layout a given bundle file is based on.
Bundles currently use the following layout version:
```shell
BundleFileVersionMajor=0
BundleFileVersionMinor=9
```

Future versions of GraalVM might alter or extend the internal structure of bundle files.
The versioning enables us to evolve the bundle format with backwards compatibility in mind.

### input

This directory contains all input data that gets passed to the `native-image` builder. 
The file _input/stage/build.json_ holds the original command line that was passed to `native-image` when the bundle was created.

Parameters that make no sense to get reapplied in a bundle-build are already filtered out.
These include:
* `--bundle-{create,apply}`
* `--verbose`
* `--dry-run`

The state of environment variables that are relevant for the build are captured in _input/stage/environment.json_.
For every `-E` argument that were seen when the bundle was created, a snapshot of its key-value pair is recorded in the file.
The remaining files _path_canonicalizations.json_ and _path_substitutions.json_ contain a record of the file-path transformations that were performed by the `native-image` tool based on the input file paths as specified by the original command line arguments.

### output

If a native executable is built as part of building the bundle (for example, the `--dry-run` option was not used), you also have an _output_ directory in the bundle.
It contains the executable that was built along with any other files that were generated as part of building.
Most output files are located in the directory _output/default_ (the executable, its debug info, and debug sources).
Builder output files, that would have been written to arbitrary absolute paths if the executable had not been built in the bundle mode, can be found in _output/other_.

### Related Documentation

* [Native Image Build Configuration](BuildConfiguration.md)
* [Native Image Build Output](BuildOutput.md)

# Native Image C API

Native Image provides a GraalVM-specific API to manage Java objects from the C/C++ languages, initialize isolates and attach threads.
The C API is available when Native Image is built as a shared library and its declarations are included in the header file that is generated during the native image build.

```c
/*
 * Structure representing an isolate. A pointer to such a structure can be
 * passed to an entry point as the execution context.
 */
struct __graal_isolate_t;
typedef struct _graal_isolate_t graal_isolate_t;

/*
 * Structure representing a thread that is attached to an isolate. A pointer to
 * such a structure can be passed to an entry point as the execution context,
 * requiring that the calling thread has been attached to that isolate.
 */
struct __graal_isolatethread_t;
typedef struct __graal_isolatethread_t graal_isolatethread_t;

/* Parameters for the creation of a new isolate. */
struct __graal_create_isolate_params_t {
    /* for future use */
};
typedef struct __graal_create_isolate_params_t graal_create_isolate_params_t;

/*
 * Create a new isolate, considering the passed parameters (which may be NULL).
 * Returns 0 on success, or a non-zero value on failure.
 * On success, the current thread is attached to the created isolate, and the
 * address of the isolate and the isolate thread structures is written to the
 * passed pointers if they are not NULL.
 */
int graal_create_isolate(graal_create_isolate_params_t* params, graal_isolate_t** isolate, graal_isolatethread_t** thread);

/*
 * Attaches the current thread to the passed isolate.
 * On failure, returns a non-zero value. On success, writes the address of the
 * created isolate thread structure to the passed pointer and returns 0.
 * If the thread has already been attached, the call succeeds and also provides
 * the thread's isolate thread structure.
 */
int graal_attach_thread(graal_isolate_t* isolate, graal_isolatethread_t** thread);

/*
 * Given an isolate to which the current thread is attached, returns the address of
 * the thread's associated isolate thread structure.  If the current thread is not
 * attached to the passed isolate or if another error occurs, returns NULL.
 */
graal_isolatethread_t* graal_get_current_thread(graal_isolate_t* isolate);

/*
 * Given an isolate thread structure, determines to which isolate it belongs and
 * returns the address of its isolate structure. If an error occurs, returns NULL
 * instead.
 */
graal_isolate_t* graal_get_isolate(graal_isolatethread_t* thread);

/*
 * Detaches the passed isolate thread from its isolate and discards any state or
 * context that is associated with it. At the time of the call, no code may still
 * be executing in the isolate thread's context.
 * Returns 0 on success, or a non-zero value on failure.
 */
int graal_detach_thread(graal_isolatethread_t* thread);

/*
 * Tears down the isolate of the passed (and still attached) isolate thread
 * waiting for any attached threads to detach from it, then discards its objects,
 * threads, and any other state or context that is associated with it.
 * Returns 0 on success, or a non-zero value on failure.
 */
int graal_tear_down_isolate(graal_isolatethread_t* thread);
```

In addition to the C level API, you can use the [JNI Invocation API](JNIInvocationAPI.md) to create an isolate from Java, expose and call Java methods embedded in a native shared library.

### Related Documentation

- [Build a Native Shared Library](guides/build-native-shared-library.md)
- [Interoperability with Native Code](InteropWithNativeCode.md)
- [JNI Invocation API](JNIInvocationAPI.md)

# Certificate Management in Native Image

Native Image provides multiple ways to specify the certificate file used to define the default TrustStore.
In the following sections we describe the available build-time and run-time options.
Note: The default behavior for `native-image` is to capture and use the default TrustStore from the build-time host environment.

## Build-time Options

During the image building process, the `native-image` builder captures the host environment's default TrustStore and embeds it into the native executable.
This TrustStore is by default created from the root certificate file provided within the JDK, but can be changed to use a different certificate file by setting the build-time system property `javax.net.ssl.trustStore` (see [Properties](guides/use-system-properties.md) for how to do it).

Since the contents of the build-time certificate file is embedded into the native executable, the file itself does not need to be present in the target environment.

## Runtime Options

The certificate file can also be changed dynamically at run time via setting the `javax.net.ssl.trustStore\*` system properties.

If any of the following system properties are set during the image execution, `native-image` also requires `javax.net.ssl.trustStore` to be set, and for it to point to an accessible certificate file:
- `javax.net.ssl.trustStore`
- `javax.net.ssl.trustStoreType`
- `javax.net.ssl.trustStoreProvider`
- `javax.net.ssl.trustStorePassword`

If any of these properties are set and `javax.net.ssl.trustStore` does not point to an accessible file, then an `UnsupportedFeatureError` will be thrown.

Note that this behavior is different than OpenJDK.
When the `javax.net.ssl.trustStore` system property is unset or invalid, OpenJDK will fallback to using a certificate file shipped within the JDK.
However, such files will not be present alongside the image executable and hence cannot be used as a fallback.

During the execution, it also possible to dynamically change the `javax.net.ssl.trustStore\*` properties and for the default TrustStore to be updated accordingly.

Finally, whenever all of the `javax.net.ssl.trustStore\*` system properties listed above are unset, the default TrustStore will be the one captured during the build time, as described in the [prior section](#build-time-options).

## Untrusted Certificates

During the image building process, a list of untrusted certificates is loaded from the file `<java.home>/lib/security/blacklisted.certs`.
This file is used when validating certificates at both build time and run time.
In other words, when a new certificate file is specified at run time via setting the `javax.net.ssl.trustStore\*` system properties, the new certificates will still be checked against the `<java.home>/lib/security/blacklisted.certs` loaded at
image build time.

# Class Initialization in Native Image

The semantics of Java requires that a class is initialized the first time it is accessed at runtime.
Class initialization has negative consequences for compiling Java applications ahead-of-time for the following two reasons:

* It significantly degrades the performance of a native executable: every access to a class (via a field or method) requires a check to ensure the class is already initialized. Without optimization, this can reduce performance by more than twofold.
* It increases the amount of computation--and time--to startup an application. For example, the simple "Hello, World!" application requires more than 300 classes to be initialized.

To reduce the negative impact of class initialization, Native Image supports class initialization at build time: it can initialize classes when it builds an executable, making runtime initialization and checks unnecessary.
All the static state from initialized classes is stored in the executable.
Access to a class's static fields that were initialized at build time is transparent to the application and works as if the class was initialized at runtime.

However, Java class initialization semantics impose several constraints that complicate class initialization policies, such as:

* When a class is initialized, all its superclasses and superinterfaces with default methods must also be initialized.
Interfaces without default methods, however, are not initialized.
To accommodate this requirement, a short-term "relevant supertype" is used, as well as a "relevant subtype" for subtypes of classes and interfaces with default methods.

* Relevant supertypes of types initialized at build time must also be initialized at build time.
* Relevant subtypes of types initialized at runtime must also be initialized at runtime.
* No instances of classes that are initialized at runtime must be present in the executable.

To enjoy the complete out-of-the-box experience of Native Image and still get the benefits of build-time initialization, Native Image does two things:

* [Build-Time Initialization](#build-time-initialization)
* [Automatic Initialization of Safe Classes](#automatic-initialization-of-safe-classes)

To track which classes were initialized and why, pass the command-line option `-H:+PrintClassInitialization` to the `native-image` tool.
This option helps you configure the `native image` builder to work as required.
The goal is to have as many classes as possible initialized at build time, yet keep the correct semantics of the application.

## Build-Time Initialization

Native Image initializes most JDK classes at build time, including the garbage collector, important JDK classes, and the deoptimizer.
For all of the classes that are initialized at build time, Native Image gives proper support so that the semantics remain consistent despite class initialization occurring at build time.
If you discover an issue with a JDK class behaving incorrectly because of class initialization at build time, please [report an issue](https://github.com/oracle/graal/issues/new).


## Automatic Initialization of Safe Classes

For application classes, Native Image tries to find classes that can be safely initialized at build time.
A class is considered safe if all of its relevant supertypes are safe and if the class initializer does not call any unsafe methods or initialize other unsafe classes.

A method is considered unsafe if:

* It transitively calls into native code (such as `System.out.println`): native code is not analyzed so Native Image cannot know if illegal actions are performed.
* It calls a method that cannot be reduced to a single target (a virtual method).
This restriction avoids the explosion of search space for the safety analysis of static initializers.
* It is substituted by Native Image. Running initializers of substituted methods would yield different results in the hosting Java virtual machine (VM) than in the produced executable.
As a result, the safety analysis would consider some methods safe but calling them would lead to illegal states.

A test that shows examples of classes that are proven safe can be found [here](https://github.com/oracle/graal/blob/master/substratevm/src/com.oracle.svm.test/src/com/oracle/svm/test/clinit/TestClassInitializationMustBeSafeEarly.java).
The list of all classes that are proven safe is output to a file via the `-H:+PrintClassInitialization` command-line option to the `native-image` tool.

> Note: You can also [Specify Class Initialization Explicitly](guides/specify-class-initialization.md).

### Related Documentation

- [Native Image Basics](NativeImageBasics.md#image-build-time-vs-image-run-time)
- [Native Image Compatibility Guide](Compatibility.md)
- [Specify Class Initialization Explicitly](guides/specify-class-initialization.md)

# Native Image Compatibility Guide

Native Image uses a different way of compiling a Java application than the traditional Java virtual machine (VM).
It distinguishes between **build time** and **run time**.
At the image build time, the `native-image` builder performs static analysis to find all the methods that are reachable from the entry point of an application.
The builder then compiles these (and only these) methods into an executable binary.
Because of this different compilation model, a Java application can behave somewhat differently when compiled into a native image.

Native Image provides an optimization to reduce the memory footprint and startup time of an application.
This approach relies on a ["closed-world assumption"](NativeImageBasics.md#static-analysis) in which all code is known at build time. That is, no new code is loaded at run time.
As with most optimizations, not all applications are amenable to this approach.
If the `native-image` builder is unable to optimize an application at build time, it generates a so-called "fallback file" that requires a Java VM to run.
We recommend to check [Native Image Basics](NativeImageBasics.md) for a detailed description what happens with your Java application at build and run times.

## Features Requiring Metadata

To be suitable for closed-world assumption, the following Java features generally require metadata to pass to `native-image` at build time. 
This metadata ensures that a native image uses the minimum amount of space necessary.

The compatibility of Native Image with the most popular Java libraries was recently enhanced by publishing [shared reachability metadata on GitHub](https://github.com/oracle/graalvm-reachability). The users can share the burden of maintaining metadata for third-party dependencies and reuse it.
See [Reachability Metadata](ReachabilityMetadata.md) to learn more.

## Features Incompatible with Closed-World Assumption

Some Java features are not yet supported within the closed-world assumption, and if used, result in a fallback file.

### `invokedynamic` Bytecode and Method Handles

Under the closed-world assumption, all methods that are called and their call sites must be known.
The `invokedynamic`method and method handles can introduce calls at run time or change the method that is invoked.

Note that `invokedynamic` use cases generated by `javac` for, for example, Java lambda expressions and String concatenation that are supported because they do not change called methods at run time.

### Security Manager

Native Image will not allow a Java Security Manager to be enabled because this functionality has deprecated since Java 17.

## Features That May Operate Differently in a Native Image

Native Image implements some Java features differently to the Java VM.

### Signal Handlers

Registering a signal handler requires a new thread to start that handles the signal and invokes shutdown hooks.
By default, no signal handlers are registered when building a native image, unless they are registered explicitly by the user.
For example, it is not recommended to register the default signal handlers when building a shared library, but it is desirable to include signal handlers when building a native executable for containerized environments, such as Docker containers.

To register the default signal handlers, pass the `--install-exit-handlers` option to the `native-image` builder.
This option gives you the same signal handlers as a Java VM.

### Class Initializers

By default, classes are initialized at run time.
This ensures compatibility, but limits some optimizations.
For faster startup and better peak performance, it is better to initialize classes at build time.
Class initialization behavior can be specified using the options `--initialize-at-build-time` or `--initialize-at-run-time` for specific classes and packages or for all classes.
Classes that are members of the JDK class libraries are initialized by default.

**Note**: Class initialization at build time may break specific assumptions in existing code.
For example, files loaded in a class initializer may not be in the same place at build time as at run time.
Also, certain objects such as a file descriptors or running threads must not be stored in a native executable.
If such objects are reachable at build time, the `native image` builder fails with an error.

For more information, see [Class Initialization in Native Image](ClassInitialization.md).

### Finalizers

The Java base class `java.lang.Object` defines the method `finalize()`.
It is called by the garbage collector on an object when garbage collection determines that there are no more references to the object.
A subclass can override the `finalize()` method to dispose of system resources or to perform other cleanup operations.

Finalizers have been deprecated since Java SE 9.
They are complicated to implement, and have badly designed semantics.
For example, a finalizer can cause an object to be reachable again by storing a reference to it in a static field.
Therefore, finalizers are not invoked.
We recommend you replace finalizers with weak references and reference queues.

### Threads

Native Image does not implement long-deprecated methods in `java.lang.Thread` such as `Thread.stop()`.

### Unsafe Memory Access

Fields that are accessed using `sun.misc.Unsafe` need to be marked as such for the static analysis if classes are initialized at build time.
In most cases, that happens automatically: field offsets stored in `static final` fields are automatically rewritten from the hosted value (the field offset for the Java VM on which the `native image` builder is running) to the native executable value, and as part of that rewrite the field is marked as `Unsafe`-accessed.
For non-standard patterns, field offsets can be recomputed manually using the annotation `RecomputeFieldValue`.

### Debugging and Monitoring

Java has some optional specifications that a Java implementation can use for debugging and monitoring Java programs, including JVMTI.
They help you monitor the Java VM at runtime for events such as compilation, for example, which do not occur in most native images.
These interfaces are built on the assumption that Java bytecodes are available at run time, which is not the case for native images built with the closed-world optimization.
Because the `native-image` builder generates a native executable, users must use native debuggers and monitoring tools (such as GDB or VTune) rather than tools targeted for Java.
JVMTI and other bytecode-based tools are not supported with Native Image.

# Limitations on Linux AArch64 Architecture

Mostly all Native Image features are supported on Linux AArch64 architecture, except for the limitations described below.

* `-R:[+|-]WriteableCodeCache`: must be disabled.
* `--libc=<value>`: `musl` is not supported.
* `--gc=<value>`: The G1 garbage collector (`G1`) is not supported.

Find a complete list of options to the `native-image` builder [here](BuildOptions.md).

### Related Documentation

* [Class Initialization in Native Image](ClassInitialization.md)
* [Reachability Metadata](ReachabilityMetadata.md)
* [GraalVM Reachability Metadata Repository](https://github.com/oracle/graalvm-reachability)

# Debug Info Feature

To add debug information to a generated native image, provide the `-g` option to the `native-image` builder:
```shell
native-image -g Hello
```

The `-g` flag instructs `native-image` to generate debug information.
The resulting image will contain debug records in a format the GNU Debugger (GDB) understands.
Additionally, you can pass `-O0` to the builder which specifies that no compiler optimizations should be performed.
Disabling all optimizations is not required, but in general it makes the debugging experience better.

Debug information is not just useful to the debugger. It can also be used by the Linux performance profiling tools `perf` and `valgrind` to correlate execution statistics such as CPU utilization or cache misses with specific, named Java methods and even link them to individual lines of Java code in the original Java source file.

By default, debug info will only include details of some of the values of parameters and local variables.
This means that the debugger will report many parameters and local variables as being undefined. If you pass `-O0` to the builder then full debug information will be included.
If you
want more parameter and local variable information to be included when employing higher
levels of optimization (`-O1` or, the default, `-O2`) you need to pass an extra command
line flag to the `native-image` command

```shell
native-image -g -H:+SourceLevelDebug Hello
```

Enabling debuginfo with flag `-g` does not make any difference to how a generated
native image is compiled and does not affect how fast it executes nor how much memory it uses at runtime.
However, it can significantly increase the size of the generated image on disk. Enabling full parameter
and local variable information by passing flag `-H:+SourceLevelDebug` can cause a program to be compiled
slightly differently and for some applications this can slow down execution.

The basic `perf report` command, which displays a histogram showing percentage execution time in each Java method, only requires passing flags `-g` and `-H:+SourceLevelDebug` to the `native-image` command.
However, more sophisticated uses of `perf` (i.e. `perf annotate`) and use of
`valgrind` requires debug info to be supplemented with linkage symbols identifying compiled Java methods.
Java method symbols are omitted from the generated native image by default but they can be retained achieved by passing one extra flag to the `native-image` command

```shell
native-image -g -H:+SourceLevelDebug -H:-DeleteLocalSymbols Hello
```

Use of this flag will result in a small increase in the size of the
resulting image file.

> Note: Native Image debugging currently works on Linux with initial support for macOS. The feature is experimental.

> Note: Debug info support for `perf` and `valgrind` on Linux is an experimental feature.

### Table of Contents

- [Source File Caching](#source-file-caching)
- [Special Considerations for Debugging Java from GDB](#special-considerations-for-debugging-java-from-gdb)
- [Identifying Source Code Location](#identifying-source-code-location)
- [Configuring Source Paths in GNU Debugger](#configuring-source-paths-in-gnu-debugger)
- [Checking Debug Info on Linux](#checking-debug-info-on-linux)
- [Debugging with Isolates](#debugging-with-isolates)
- [Debugging Helper Methods](#debugging-helper-methods)
- [Special Considerations for using perf and valgrind](#special-considerations-for-using-perf-and-valgrind)

## Source File Caching

The `-g` option also enables caching of sources for any JDK runtime classes, GraalVM classes, and application classes which can be located when generating a native executable.
By default, the cache is created alongside the generated binary in a subdirectory named `sources`.
If a target directory for the native executable is specified using option `-H:Path=...` then the cache is also relocated under that same target. 
Use a command line option to provide an alternative path to `sources` and to configure source file search path roots for the debugger.
Files in the cache are located in a directory hierarchy that matches the file path information included in the debug records of the native executable.
The source cache should contain all the files needed to debug the generated binary and nothing more.
This local cache provides a convenient way of making just the necessary sources available to the debugger or IDE when debugging a native executable.

The implementation tries to be smart about locating source files.
It uses the current `JAVA_HOME` to locate the JDK src.zip when searching for JDK runtime sources.
It also uses entries in the classpath to suggest locations for GraalVM source files and application source files (see below for precise details of the scheme used to identify source locations).
However, source layouts do vary and it may not be possible to find all sources.
Hence, users can specify the location of source files explicitly on the command line using option `DebugInfoSourceSearchPath`:

```shell
javac --source-path apps/greeter/src \
    -d apps/greeter/classes org/my/greeter/*Greeter.java
javac -cp apps/greeter/classes \
    --source-path apps/hello/src \
    -d apps/hello/classes org/my/hello/Hello.java
native-image -g \
    -H:-SpawnIsolates \
    -H:DebugInfoSourceSearchPath=apps/hello/src \
    -H:DebugInfoSourceSearchPath=apps/greeter/src \
    -cp apps/hello/classes:apps/greeter/classes org.my.hello.Hello
```

The `DebugInfoSourceSearchPath` option can be repeated as many times as required to notify all the target source locations.
The value passed to this option can be either an absolute or relative path.
It can identify either a directory, a source JAR, or a source ZIP file.
It is also possible to specify several source roots at once using a comma separator:

```shell
native-image -g \
    -H:DebugInfoSourceSearchPath=apps/hello/target/hello-sources.jar,apps/greeter/target/greeter-sources.jar \
    -cp apps/target/hello.jar:apps/target/greeter.jar \
    org.my.Hello
```

By default, the cache of application, GraalVM, and JDK sources is created in a directory named `sources`.
The `DebugInfoSourceCacheRoot` option can be used to specify an alternative path, which can be absolute or relative.
In the latter case the path is interpreted relative to the target directory for the generated executable specified via option `-H:Path` (which defaults to the current working directory).
As an example, the following variant of the previous command specifies an absolute temporary directory path constructed using the current process `id`:

```shell
SOURCE_CACHE_ROOT=/tmp/$$/sources
native-image -g \
    -H:-SpawnIsolates \
    -H:DebugInfoSourceCacheRoot=$SOURCE_CACHE_ROOT \
    -H:DebugInfoSourceSearchPath=apps/hello/target/hello-sources.jar,apps/greeter/target/greeter-sources.jar \
    -cp apps/target/hello.jar:apps/target/greeter.jar \
    org.my.Hello
```
The resulting cache directory will be something like `/tmp/1272696/sources`.

If the source cache path includes a directory that does not yet exist, it will be created during population of the cache.

Note that in all the examples above the `DebugInfoSourceSearchPath` options are actually redundant.
In the first case, the classpath entries for _apps/hello/classes_ and _apps/greeter/classes_ will be used to derive the default search roots _apps/hello/src_ and _apps/greeter/src_.
In the second case, the classpath entries for _apps/target/hello.jar_ and _apps/target/greeter.jar_ will be used to derive the default search roots _apps/target/hello-sources.jar_ and _apps/target/greeter-sources.jar_.

## Supported Features

The currently supported features include:

  - break points configured by file and line, or by method name
  - single stepping by line including both into and over function calls
  - stack backtraces (not including frames detailing inlined code)
  - printing of primitive values
  - structured (field by field) printing of Java objects
  - casting/printing objects at different levels of generality
  - access through object networks via path expressions
  - reference by name to methods and static field data
  - reference by name to values bound to parameter and local vars
  - reference by name to class constants

Note that single stepping within a compiled method includes file and line number info for inlined code, including inlined GraalVM methods.
So, GDB may switch files even though you are still in the same compiled method.

### Special considerations for debugging Java from GDB

GDB does not currently include support for Java debugging.
In consequence, debug capability has been implemented by generating debug info that models the Java program as an equivalent C++ program. 
Java class, array and interface references are actually pointers to records that contain the relevant field/array data.
In the corresponding C++ model the Java name is used to label the underlying C++ (class/struct) layout types and Java references appear as pointers.

So, for example in the DWARF debug info model `java.lang.String` identifies a C++ class.
This class layout type declares the expected fields like `hash` of type `int` and `value` of type `byte[]` and methods like `String(byte[])`, `charAt(int)`, etc. However, the copy constructor which appears in Java as `String(String)` appears in `gdb` with the signature `String(java.lang.String *)`.

The C++ layout class inherits fields and methods from class (layout) type `java.lang.Object` using C++ public inheritance.
The latter in turn inherits standard oop (ordinary object pointer) header fields from a special struct class named `_objhdr` which includes two fields. The first field is called
`hub` and its type is `java.lang.Class *` i.e. it is a pointer to the object's
class. The second field is called `idHash` and has type `int`. It stores an
identity hashcode for the object.

The `ptype` command can be used to print details of a specific type.
Note that the Java type name must be specified in quotes because to escape the embedded `.` characters.

```
(gdb) ptype 'java.lang.String'
type = class java.lang.String : public java.lang.Object {
  private:
    byte [] *value;
    int hash;
    byte coder;

  public:
    void String(byte [] *);
    void String(char [] *);
    void String(byte [] *, java.lang.String *);
    . . .
    char charAt(int);
    . . .
    java.lang.String * concat(java.lang.String *);
    . . .
}
```

The ptype command can also be used to identify the static type of a Java
data value. The current example session is for a simple hello world
program. Main method `Hello.main` is passed a single parameter
`args` whose Java type is `String[]`. If the debugger is stopped at
entry to `main` we can use `ptype` to print the type of `args`.
 
 ```
(gdb) ptype args
type = class java.lang.String[] : public java.lang.Object {
  public:
    int len;
    java.lang.String *data[0];
} *
```

There are a few details worth highlighting here. Firstly, the debugger
sees a Java array reference as a pointer type, as it does every Java object
reference.

Secondly, the pointer points to a structure, actually a C++ class,
that models the layout of the Java array using an integer length field
and a data field whose type is a C++ array embedded into the block of
memory that models the array object.

Elements of the array data field are references to the base type, in
this case pointers to `java.lang.String`. The data array has a nominal
length of 0. However, the block of memory allocated for the `String[]`
object actually includes enough space to hold the number of pointers
determined by the value of field `len`.

Finally, notice that the C++ class `java.lang.String[]` inherits from
the C++ class `java.lang.Object`. So, an array is still also an object.
In particular, as we will see when we print the object contents, this
means that every array also includes the object header fields that all
Java objects share.

The print command can be used to display the object reference as a memory
address. 

```
(gdb) print args
$1 = (java.lang.String[] *) 0x7ffff7c01130
```

It can also be used to print the contents of the object field by field. This
is achieved by dereferencing the pointer using the `*` operator.

```
(gdb) print *args
$2 = {
  <java.lang.Object> = {
    <_objhdr> = {
      hub = 0xaa90f0,
      idHash = 0
    }, <No data fields>}, 
  members of java.lang.String[]:
  len = 1,
  data = 0x7ffff7c01140
}
```

The array object contains embedded fields inherited from class
`_objhdr` via parent class `Object`. `_objhdr` is a synthetic type
added to the deubg info to model fields that are present at the start
of all objects. They include `hub` which is a reference to the object's
class and `hashId` a unique numeric hash code.

Clearly, the debugger knows the type (`java.lang.String[]`) and location
in memory (`0x7ffff7c010b8`) of local variable `args`. It also knows about
the layout of the fields embedded in the referenced object. This means
it is possible to use the C++ `.` and `->` operators in debugger commands
to traverse the underlying object data structures.

```
(gdb) print args->data[0]
$3 = (java.lang.String *) 0x7ffff7c01160
(gdb) print *args->data[0]
$4 = {
   <java.lang.Object> = {
     <_objhdr> = {
      hub = 0xaa3350
     }, <No data fields>},
   members of java.lang.String:
   value = 0x7ffff7c01180,
   hash = 0,
   coder = 0 '\000'
 }
(gdb) print *args->data[0]->value
$5 = {
  <java.lang.Object> = {
    <_objhdr> = {
      hub = 0xaa3068,
      idHash = 0
    }, <No data fields>}, 
  members of byte []:
  len = 6,
  data = 0x7ffff7c01190 "Andrew"
}
 ```

Returning to the `hub` field in the object header it was
mentioned before that this is actually a reference to the object's
class. This is actually an instance of Java type `java.lang.Class`.
Note that the field is typed by gdb using a pointer
to the underlying C++ class (layout) type.

```
(gdb) print args->hub
$6 = (java.lang.Class *) 0xaa90f0
```

All classes, from Object downwards inherit from a common, automatically generated header type `_objhdr`.
It is this header type which includes the `hub` field:

```
(gdb) ptype _objhdr
type = struct _objhdr {
    java.lang.Class *hub;
    int idHash;
}

(gdb) ptype 'java.lang.Object'
type = class java.lang.Object : public _objhdr {
  public:
    void Object(void);
    . . .
```

The fact that all objects have a common header pointing to a class
makes it possible to perform a simple test to decide if an address
is an object reference and, if so,  what the object's class is.
Given a valid object reference it is always possible to print the
contents of the `String` referenced from the `hub`'s name field.

Note that as a consequence, this allows every object observed by the debugger
to be downcast to its dynamic type. i.e. even if the debugger only sees the static
type of e.g. java.nio.file.Path we can easily downcast to the dynamic type, which
might be a subtype such as `jdk.nio.zipfs.ZipPath`, thus making it possible to inspect
fields that we would not be able to observe from the static type alone.
First the value is cast to an object reference.
Then a path expression is used to dereference through the the `hub` field and the `hub`'s name field to the `byte[]` value array located in the name `String`.

```
(gdb) print/x ((_objhdr *)$rdi)
$7 = (_objhdr *) 0x7ffff7c01130
(gdb) print *$7->hub->name->value
$8 = {
  <java.lang.Object> = {
    <_objhdr> = {
      hub = 0xaa3068,
      idHash = 178613527
    }, <No data fields>}, 
   members of byte []:
   len = 19,
  data = 0x8779c8 "[Ljava.lang.String;"
 }
```

The value in register `rdi` is obviously a reference to a String array.
Indeed, this is no coincidence. The example session has stopped at a break
point placed at the entry to `Hello.main` and at that point the value for
the `String[]` parameter `args` will be located in register `rdi`. Looking
back we can see that the value in `rdi` is the same value as was printed by
command `print args`. 

A simpler command which allows just the name of the `hub` object to be printed is as follows:

```
(gdb) x/s $7->hub->name->value->data
798:	"[Ljava.lang.String;"
```

Indeed it is useful to define a `gdb` command `hubname_raw` to execute this operation on an arbitrary raw memory address.

```
define hubname_raw
  x/s (('java.lang.Object' *)($arg0))->hub->name->value->data
end

(gdb) hubname_raw $rdi
0x8779c8:	"[Ljava.lang.String;"
```

Attempting to print the hub name for an invalid reference will fail
safe, printing an error message.

```
(gdb) p/x $rdx
$5 = 0x2
(gdb) hubname $rdx
Cannot access memory at address 0x2
```

If `gdb` already knows the Java type for a reference it can be printed without casting using a simpler version of the hubname command.
For example, the String array retrieved above as `$1` has a known type.

```
(gdb) ptype $1
type = class java.lang.String[] : public java.lang.Object {
    int len;
    java.lang.String *data[0];
} *

define hubname
  x/s (($arg0))->hub->name->value->data
end

(gdb) hubname $1
0x8779c8:	"[Ljava.lang.String;"
```

The native image heap contains a unique hub object (i.e. instance of
`java.lang.Class`) for every Java type that is included in the
image. It is possible to refer to these class constants using the
standard Java class literal syntax:

```
(gdb) print 'Hello.class'
$6 = {
  <java.lang.Object> = {
    <_objhdr> = {
      hub = 0xaabd00,
      idHash = 1589947226
    }, <No data fields>}, 
  members of java.lang.Class:
  typeCheckStart = 13,
  name = 0xbd57f0,
  ...
```

Unfortunately it is necessary to quote the class constant literal to
avoid gdb interpreting the embedded `.` character as a field access.

Note that the type of a class constant literal is `java.lang.Class`
rather than `java.lang.Class *`.

Class constants exist for Java instance classes, interfaces, array
classes and arrays, including primitive arrays:

```
(gdb)  print 'java.util.List.class'.name
$7 = (java.lang.String *) 0xb1f698
(gdb) print 'java.lang.String[].class'.name->value->data
$8 = 0x8e6d78 "[Ljava.lang.String;"
(gdb) print 'long.class'.name->value->data
$9 = 0xc87b78 "long"
(gdb) x/s  'byte[].class'.name->value->data
0x925a00:	"[B"
(gdb) 
```

Interface layouts are modelled as C++ union types.
The members of the union include the C++ layout types for all Java classes which implement the interface.

```
(gdb) ptype 'java.lang.CharSequence'
type = union java.lang.CharSequence {
    java.nio.CharBuffer _java.nio.CharBuffer;
    java.lang.AbstractStringBuilder _java.lang.AbstractStringBuilder;
    java.lang.String _java.lang.String;
    java.lang.StringBuilder _java.lang.StringBuilder;
    java.lang.StringBuffer _java.lang.StringBuffer;
}
```

Given a reference typed to an interface it can be resolved to the relevant class type by viewing it through the relevant union element.

If we take the first String in the args array we can ask `gdb` to cast it to interface `CharSequence`.

```
(gdb) print args->data[0]
$10 = (java.lang.String *) 0x7ffff7c01160
(gdb) print ('java.lang.CharSequence' *)$10
$11 = (java.lang.CharSequence *) 0x7ffff7c01160
```

The `hubname` command will not work with this union type because it is only objects of the elements of the union that include the `hub` field:

```
(gdb) hubname $11
There is no member named hub.
```

However, since all elements include the same header any one of them can be passed to `hubname` in order to identify the actual type.
This allows the correct union element to be selected:

```
(gdb) hubname $11->'_java.nio.CharBuffer'
0x95cc58:	"java.lang.String`\302\236"
(gdb) print $11->'_java.lang.String'
$12 = {
  <java.lang.Object> = {
    <_objhdr> = {
      hub = 0xaa3350,
      idHash = 0
    }, <No data fields>},
  members of java.lang.String:
  hash = 0,
  value = 0x7ffff7c01180,
  coder = 0 '\000'
}
```

Notice that the printed class name for `hub` includes some trailing characters.
That is because a data array storing Java String text is not guaranteed to be zero-terminated.

The debugger does not just understand the name and type of local and
parameter variables. It also knows about method names and static field
names.

The following command places a breakpoint on the main entry point for class `Hello`.
Note that since GDB thinks this is a C++ method it uses the `::` separator to separate the method name from the class name.

```
(gdb) info func ::main
All functions matching regular expression "::main":

File Hello.java:
	void Hello::main(java.lang.String[] *);
(gdb) x/4i Hello::main
=> 0x4065a0 <Hello::main(java.lang.String[] *)>:	sub    $0x8,%rsp
   0x4065a4 <Hello::main(java.lang.String[] *)+4>:	cmp    0x8(%r15),%rsp
   0x4065a8 <Hello::main(java.lang.String[] *)+8>:	jbe    0x4065fd <Hello::main(java.lang.String[] *)+93>
   0x4065ae <Hello::main(java.lang.String[] *)+14>:	callq  0x406050 <Hello$Greeter::greeter(java.lang.String[] *)>
(gdb) b Hello::main
Breakpoint 1 at 0x4065a0: file Hello.java, line 43.
```

An example of a static field containing Object data is provided by the static field `powerCache` in class `BigInteger`.

```
(gdb) ptype 'java.math.BigInteger'
type = class _java.math.BigInteger : public _java.lang.Number {
  public:
    int [] mag;
    int signum;
  private:
    int bitLengthPlusOne;
    int lowestSetBitPlusTwo;
    int firstNonzeroIntNumPlusTwo;
    static java.math.BigInteger[][] powerCache;
    . . .
  public:
    void BigInteger(byte [] *);
    void BigInteger(java.lang.String *, int);
    . . .
}
(gdb) info var powerCache
All variables matching regular expression "powerCache":

File java/math/BigInteger.java:
	java.math.BigInteger[][] *java.math.BigInteger::powerCache;
```

The static variable name can be used to refer to the value stored in this field.
Note also that the address operator can be used identify the location (address) of the field in the heap.

```
(gdb) p 'java.math.BigInteger'::powerCache
$13 = (java.math.BigInteger[][] *) 0xced5f8
(gdb) p &'java.math.BigInteger'::powerCache
$14 = (java.math.BigInteger[][] **) 0xced3f0
```

The debugger dereferences through symbolic names for static fields to access the primitive value or object stored in the field.

```
(gdb) p *'java.math.BigInteger'::powerCache
$15 = {
  <java.lang.Object> = {
    <_objhdr> = {
    hub = 0xb8dc70,
    idHash = 1669655018
    }, <No data fields>},
  members of _java.math.BigInteger[][]:
  len = 37,
  data = 0xced608
}
(gdb) p 'java.math.BigInteger'::powerCache->data[0]@4
$16 = {0x0, 0x0, 0xed5780, 0xed5768}
(gdb) p *'java.math.BigInteger'::powerCache->data[2]
$17 = {
  <java.lang.Object> = {
    <_objhdr> = {
    hub = 0xabea50,
    idHash = 289329064
    }, <No data fields>},
  members of java.math.BigInteger[]:
  len = 1,
  data = 0xed5790
}
(gdb) p *'java.math.BigInteger'::powerCache->data[2]->data[0]
$18 = {
  <java.lang.Number> = {
    <java.lang.Object> = {
      <_objhdr> = {
        hub = 0xabed80
      }, <No data fields>}, <No data fields>},
  members of java.math.BigInteger:
  mag = 0xcbc648,
  signum = 1,
  bitLengthPlusOne = 0,
  lowestSetBitPlusTwo = 0,
  firstNonzeroIntNumPlusTwo = 0
}
```

## Identifying Source Code Location

One goal of the implementation is to make it simple to configure the debugger so that it can identify the relevant source file when it stops during program execution. The `native-image` tool tries to achieve this by accumulating the relevant sources in a suitably structured file cache.

The `native-image` tool uses different strategies to locate source files for JDK runtime classes, GraalVM classes, and application source classes for inclusion in the local sources cache.
It identifies which strategy to use based on the package name of the class.
So, for example, packages starting with `java.*` or `jdk.*` are JDK classes; packages starting with `org.graal.*` or `com.oracle.svm.*` are GraalVM classes; any other packages are regarded as application classes.

Sources for JDK runtime classes are retrieved from the _src.zip_ found in the JDK release used to run the native image generation process.
Retrieved files are cached under subdirectory _sources_, using the module name (for JDK11) and package name of the associated class to define the directory hierarchy in which the source is located.

For example, on Linux the source for `class java.util.HashMap` will be cached in file _sources/java.base/java/util/HashMap.java_.
Debug info records for this class and its methods will identify this source file using the relative directory path _java.base/java/util_ and file name _HashMap.java_. On Windows things will be the same modulo use of `\` rather than `/` as the file separator.

Sources for GraalVM classes are retrieved from ZIP files or source directories derived from entries in the classpath.
Retrieved files are cached under subdirectory _sources_, using the package name of the associated class to define the directory hierarchy in which the source is located (e.g., class `com.oracle.svm.core.VM` has its source file cached at `sources/com/oracle/svm/core/VM.java`).

The lookup scheme for cached GraalVM sources varies depending upon what is found in each classpath entry.
Given a JAR file entry like _/path/to/foo.jar_, the corresponding file _/path/to/foo.src.zip_ is considered as a candidate ZIP file system from which source files may be extracted.
When the entry specifies a directory like _/path/to/bar_, then directories _/path/to/bar/src_ and _/path/to/bar/src_gen_ are considered as candidates.
Candidates are skipped when the ZIP file or source directory does not exist, or it does not contain at least one subdirectory hierarchy that matches one of the the expected GraalVM package hierarchies.

Sources for application classes are retrieved from source JAR files or source directories derived from entries in the classpath.
Retrieved files are cached under subdirectory _sources_, using the package name of the associated class to define the directory hierarchy in which the source is located (e.g., class `org.my.foo.Foo` has its source file cached as `sources/org/my/foo/Foo.java`).

The lookup scheme for cached application sources varies depending upon what is found in each classpath entry.
Given a JAR file entry like _/path/to/foo.jar_, the corresponding JAR _/path/to/foo-sources.jar_ is considered as a candidate ZIP file system from which source files may be extracted.
When the entry specifies a dir like _/path/to/bar/classes_ or _/path/to/bar/target/classes_ then one of the directories
_/path/to/bar/src/main/java_, _/path/to/bar/src/java_ or _/path/to/bar/src_ is selected as a candidate (in that order of preference).
Finally, the current directory in which the native executable is being run is also considered as a candidate.

These lookup strategies are only provisional and may need extending in the future.
However, it is possible to make missing sources available by other means.
One option is to unzip extra app source JAR files, or copy extra app source trees into the cache.
Another is to configure extra source search paths.

## Configuring Source Paths in GNU Debugger

By default, GDB will employ the local directory root `sources` to locate the source files for your application classes, GraalVM classes, and JDK runtime classes.
If the sources cache is not located in the directory in which you run GDB, you can configure the required paths using the following command:

```
(gdb) set directories /path/to/sources/
```

The argument to the set directories command should identify the location of the sources cache as an absolute path or a relative path from the working directory of the `gdb` session.

Note that the current implementation does not yet find some sources for the GraalVM JIT compiler in the _org.graalvm.compiler*_ package subspace.

You can supplement the files cached in `sources` by unzipping application source JAR files or copying application source trees into the cache.
You will need to ensure that any new subdirectory you add to `sources` corresponds to the top level package for the classes whose sources are being included.

You can also add extra directories to the search path using the `set directories` command:
```shell
(gdb) set directories /path/to/my/sources/:/path/to/my/other/sources
```
Note that the GNU Debugger does not understand ZIP format file systems so any extra entries you add must identify a directory tree containing the relevant sources.
Once again, top level entries in the directory added to the search path must correspond to the top level package for the classes whose sources are being included.

## Checking Debug Info on Linux

Note that this is only of interest to those who want to understand how the debug info implementation works or want to troubleshoot problems encountered during debugging that might relate to the debug info encoding.

The `objdump` command can be used to display the debug info embedded into a native executable.
The following commands (which all assume the target binary is called `hello`) can be used to display all generated content:
```
objdump --dwarf=info hello > info
objdump --dwarf=abbrev hello > abbrev
objdump --dwarf=ranges hello > ranges
objdump --dwarf=decodedline hello > decodedline
objdump --dwarf=rawline hello > rawline
objdump --dwarf=str hello > str
objdump --dwarf=loc hello > loc
objdump --dwarf=frames hello > frames
```

The *info* section includes details of all compiled Java methods.

The *abbrev* section defines the layout of records in the info section that describe Java files (compilation units) and methods.

The *ranges* section details the start and end addresses of method code segments.

The *decodedline* section maps subsegments of method code range segments to files and line numbers.
This mapping includes entries for files and line numbers for inlined methods.

The *rawline* segment provides details of how the line table is generated using DWARF state machine instructions that encode file, line, and address transitions.

The *loc* section provides details of address ranges within
which parameter and local variables declared in the info section
are known to have a determinate value. The details identify where
the value is located, either in a machine register, on the stack or
at a specific address in memory.

The *str* section provides a lookup table for strings referenced from records in the info section.

The *frames* section lists transition points in compiled methods where a (fixed size) stack frame is pushed or popped, allowing the debugger to identify each frame's current and previous stack pointers and its return address.

Note that some of the content embedded in the debug records is generated by the C compiler and belongs to code that is either in libraries or the C lib bootstrap code that is bundled in with the Java method code.

### Currently Supported Targets

The prototype is currently implemented only for the GNU Debugger on Linux:

  - Linux/x86_64 support has been tested and should work correctly

  - Linux/AArch64 support is present but has not yet been fully verified (break points should work ok but stack backtraces may be incorrect)

Windows support is still under development.

## Debugging with Isolates

Enabling the use of [isolates](https://medium.com/graalvm/isolates-and-compressed-references-more-flexible-and-efficient-memory-management-for-graalvm-a044cc50b67e), by passing command line option `-H:-SpawnIsolates` to the `native-image` builder, affects the way ordinary object pointers (oops) are encoded.
In turn, that means the debug info generator has to provide `gdb` with information about how to translate an encoded oop to the address in memory, where the object data is stored.
This sometimes requires care when asking `gdb` to process encoded oops vs decoded raw addresses.

When isolates are disabled, oops are essentially raw addresses pointing directly at the object contents.
This is generally the same whether the oop is embedded in a static/instance field or is referenced from a local or parameter variable located in a register or saved to the stack.
It is not quite that simple because the bottom 3 bits of some oops may be used to hold "tags" that record certain transient properties of an object.
However, the debug info provided to `gdb` means that it will remove these tag bits before dereferencing the oop as an address.

By contrast, when isolates are enabled, oops references stored in static or instance fields are actually relative addresses, offsets from a dedicated heap base register (r14 on x86_64, r29 on AArch64), rather than direct addresses (in a few special cases the offset may also have some low tag bits set).
When an "indirect" oop of this kind gets loaded during execution, it is almost always immediately converted to a "raw" address by adding the offset to the heap base register value.
So, oops which occur as the value of local or parameter vars are actually raw addresses.

> Note that on some operating systems enabling isolates causes problems with printing of objects when using a `gdb` release version 10 or earlier. It is currently recommended to disable use of isolates, by passing command line option `-H:-SpawnIsolates`, when generating debug info if your operating system includes one of these earlier releases. Alternatively, you may be able to upgrade your debugger to a later version.

The DWARF info encoded into the image, when isolates are enabled, tells `gdb` to rebase indirect oops whenever it tries to dereference them to access underlying object data.
This is normally automatic and transparent, but it is visible in the underlying type model that `gdb` displays when you ask for the type of objects.

For example, consider the static field we encountered above.
Printing its type in an image that uses isolates shows that this static field has a different type to the expected one:

```
(gdb) ptype 'java.math.BigInteger'::powerCache
type = class _z_.java.math.BigInteger[][] : public java.math.BigInteger[][] {
} *
```
The field is typed as `_z_.java.math.BigInteger[][]` which is an empty wrapper class that inherits from the expected type `java.math.BigInteger[][]`.
This wrapper type is essentially the same as the original but the DWARF info record that defines it includes information that tells gdb how to convert pointers to this type.

When `gdb` is asked to print the oop stored in this field it is clear that it is an offset rather than a raw address.

```
(gdb) p/x 'java.math.BigInteger'::powerCache
$1 = 0x286c08
(gdb) x/x 0x286c08
0x286c08:	Cannot access memory at address 0x286c08
```

However, when `gdb` is asked to dereference through the field, it applies the necessary address conversion to the oop and fetches the correct data.

```
(gdb) p/x *'java.math.BigInteger'::powerCache
$2 = {
  <java.math.BigInteger[][]> = {
    <java.lang.Object> = {
      <_objhdr> = {
        hub = 0x1ec0e2,
        idHash = 0x2f462321
      }, <No data fields>},
    members of java.math.BigInteger[][]:
    len = 0x25,
    data = 0x7ffff7a86c18
  }, <No data fields>}
```

Printing the type of the `hub` field or the data array shows that they are also modelled using indirect types:

```
(gdb) ptype $1->hub
type = class _z_.java.lang.Class : public java.lang.Class {
} *
(gdb) ptype $2->data
type = class _z_.java.math.BigInteger[] : public java.math.BigInteger[] {
} *[0]
```

The debugger still knows how to dereference these oops:

```
(gdb) p $1->hub
$3 = (_z_.java.lang.Class *) 0x1ec0e2
(gdb) x/x $1->hub
0x1ec0e2:	Cannot access memory at address 0x1ec0e2
(gdb) p *$1->hub
$4 = {
  <java.lang.Class> = {
    <java.lang.Object> = {
      <_objhdr> = {
        hub = 0x1dc860,
        idHash = 1530752816
      }, <No data fields>},
    members of java.lang.Class:
    name = 0x171af8,
    . . .
  }, <No data fields>}

```

Since the indirect types inherit from the corresponding raw type it is possible to use an expression that identifies an indirect type pointer in almost all cases where an expression identifying a raw type pointer would work.
The only case case where care might be needed is when casting a displayed numeric field value or displayed register value.

For example, if the indirect `hub` oop printed above is passed to `hubname_raw`, the cast to type Object internal to that command fails to force the required indirect oops translation.
The resulting memory access fails:

```
(gdb) hubname_raw 0x1dc860
Cannot access memory at address 0x1dc860
```

In this case it is necessary to use a slightly different command that casts its argument to an indirect pointer type:
```
(gdb) define hubname_indirect
 x/s (('_z_.java.lang.Object' *)($arg0))->hub->name->value->data
end
(gdb) hubname_indirect 0x1dc860
0x7ffff78a52f0:	"java.lang.Class"
```

## Debugging Helper Methods

On platforms where the debugging information is not fully supported, or when debugging complex issues, it can be helpful to print or query high-level information about the Native Image execution state.
For those scenarios, Native Image provides debug helper methods that can be embedded into a native executable by specifying the build-time option `-H:+IncludeDebugHelperMethods`.
While debugging, it is then possible to invoke those debug helper methods like any normal C method.
This functionality is compatible with pretty much any debugger.

While debugging with gdb, the following command can be used to list all debug helper methods that are embedded into the native image:
```
(gdb) info functions svm_dbg_
```

Before invoking a method, it is best to directly look at the source code of the Java class `DebugHelper` to determine which arguments each method expects.
For example, calling the method below prints high-level information about the Native Image execution state similar to what is printed for a fatal error:
```
(gdb) call svm_dbg_print_fatalErrorDiagnostics($r15, $rsp, $rip)
```

## Special Considerations for using perf and valgrind

Debug info includes details of address ranges for top level and
inlined compiled method code as well as mappings from code addresses
to the corresponding source files and lines.
`perf` and `valgrind` are able to use this information for some of
their recording and reporting operations.
For example, `perf report` is able to associate code adresses sampled
during a `perf record` session with Java methods and print the
DWARF-derived method name for the method in its output histogram.

```
    . . .
    68.18%     0.00%  dirtest          dirtest               [.] _start
            |
            ---_start
               __libc_start_main_alias_2 (inlined)
               |          
               |--65.21%--__libc_start_call_main
               |          com.oracle.svm.core.code.IsolateEnterStub::JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b (inlined)
               |          com.oracle.svm.core.JavaMainWrapper::run (inlined)
               |          |          
               |          |--55.84%--com.oracle.svm.core.JavaMainWrapper::runCore (inlined)
               |          |          com.oracle.svm.core.JavaMainWrapper::runCore0 (inlined)
               |          |          |          
               |          |          |--55.25%--DirTest::main (inlined)
               |          |          |          |          
               |          |          |           --54.91%--DirTest::listAll (inlined)
               . . .
```              

Unfortunately, other operations require Java methods to be identified
by an ELF (local) function symbol table entry locating the start of
the compiled method code.
In particular, assembly code dumps provided by both tools identify
branch and call targets using an offset from the nearest symbol.
Omitting Java method symbols means that offsets are generally
displayed relative to some unrelated global symbol, usually the entry
point for a method exported for invocation by C code.

As an illustration of the problem, the following excerpted output from
`perf annotate` displays the first few annotated instructions of the
compiled code for method `java.lang.String::String()`.

```
    . . .
         : 501    java.lang.String::String():
         : 521    public String(byte[] bytes, int offset, int length, Charset charset) {
    0.00 :   519d50: sub    $0x68,%rsp
    0.00 :   519d54: mov    %rdi,0x38(%rsp)
    0.00 :   519d59: mov    %rsi,0x30(%rsp)
    0.00 :   519d5e: mov    %edx,0x64(%rsp)
    0.00 :   519d62: mov    %ecx,0x60(%rsp)
    0.00 :   519d66: mov    %r8,0x28(%rsp)
    0.00 :   519d6b: cmp    0x8(%r15),%rsp
    0.00 :   519d6f: jbe    51ae1a <graal_vm_locator_symbol+0xe26ba>
    0.00 :   519d75: nop
    0.00 :   519d76: nop
         : 522    Objects.requireNonNull(charset);
    0.00 :   519d77: nop
         : 524    java.util.Objects::requireNonNull():
         : 207    if (obj == null)
    0.00 :   519d78: nop
    0.00 :   519d79: nop
         : 209    return obj;
    . . .
```

The leftmost column shows percentages for the amount of time recorded
at each instruction in samples obtained during the `perf record` run.
Each instruction is prefaced with it's address in the program's code
section.
The disassembly interleaves the source lines from which the code is
derived, 521-524 for the top level code and 207-209 for the code
inlined from from `Objects.requireNonNull()`.
Also, the start of the method is labelled with the name defined in the
DWARF debug info, `java.lang.String::String()`.
However, the branch instruction `jbe` at address `0x519d6f` uses a
very large offset from `graal_vm_locator_symbol`.
The printed offset does identify the correct address relative to the
location of the symbol.
However, this fails to make clear that the target address actually
lies within the compiled code range for method `String::String()` i.e. that thsi is a method-local branch.

Readability of the tool output is significantly improved if
option `-H-DeleteLocalSymbols` is passed to the `native-image`
command.
The equivalent `perf annotate` output with this option enabled is as
follows:

```
    . . .
         : 5      000000000051aac0 <String_constructor_f60263d569497f1facccd5467ef60532e990f75d>:
         : 6      java.lang.String::String():
         : 521    *          {@code offset} is greater than {@code bytes.length - length}
         : 522    *
         : 523    * @since  1.6
         : 524    */
         : 525    @SuppressWarnings("removal")
         : 526    public String(byte[] bytes, int offset, int length, Charset charset) {
    0.00 :   51aac0: sub    $0x68,%rsp
    0.00 :   51aac4: mov    %rdi,0x38(%rsp)
    0.00 :   51aac9: mov    %rsi,0x30(%rsp)
    0.00 :   51aace: mov    %edx,0x64(%rsp)
    0.00 :   51aad2: mov    %ecx,0x60(%rsp)
    0.00 :   51aad6: mov    %r8,0x28(%rsp)
    0.00 :   51aadb: cmp    0x8(%r15),%rsp
    0.00 :   51aadf: jbe    51bbc1 <String_constructor_f60263d569497f1facccd5467ef60532e990f75d+0x1101>
    0.00 :   51aae5: nop
    0.00 :   51aae6: nop
         : 522    Objects.requireNonNull(charset);
    0.00 :   51aae7: nop
         : 524    java.util.Objects::requireNonNull():
         : 207    * @param <T> the type of the reference
         : 208    * @return {@code obj} if not {@code null}
         : 209    * @throws NullPointerException if {@code obj} is {@code null}
         : 210    */
         : 211    public static <T> T requireNonNull(T obj) {
         : 212    if (obj == null)
    0.00 :   51aae8: nop
    0.00 :   51aae9: nop
         : 209    throw new NullPointerException();
         : 210    return obj;
    . . .
```

In this version the start address of the method is now labelled with
the mangled symbol name `String_constructor_f60263d569497f1facccd5467ef60532e990f75d`
as well as the DWARF name.
The branch target is now printed using an offset from that start
symbol.

Unfortunately, `perf` and `valgrind` do not correctly understand the
mangling algorithm employed by GraalVM, nor are they currently able to
replace the mangled name with the DWARF name in the disassembly even
though both symbol and DWARF function data are known to identify code
starting at the same address.
So, the branch instruction still prints its target using a symbol plus
offset but it is at least using the method symbol this time.

Also, because address `51aac0` is now recognized as a method start,
`perf` has preceded the first line of the method with 5 context lines,
which list the tail end of the method's javadoc comment.
Unfortunately, perf has numbered these lines incorrectly, labelling
the first comment with 521 rather than 516.

Executing command `perf annotate` will provide a disassembly listing
for all methods and C functions in the image.
It is possible to annotate a specific method by passing it's name as
an argument to the perf annotate command.
Note, however, that `perf` requries the mangled symbol name as
argument rather than the DWARF name.
So, in order to annotate method `java.lang.String::String()` it is
necessary to run command `perf annotate
String_constructor_f60263d569497f1facccd5467ef60532e990f75d`.

The `valgrind` tool `callgrind` also requires local symbols to be
retained in order to provide high quality output.
When `callgrind` is used in combination with a viewer like
`kcachegrind` it is possible to identify a great deal of valuable
information about native image execution aand relate it back to
specific source code lines.

### Call-graph recording with `perf record`

Normally when perf does stack frame recording (i.e. when `--call-graph` is used), it uses frame pointers to recognize the individual stack frames.
This assumes that the executable that gets profiled actually preserves frame pointers whenever a function gets called.
For native images, this can be achieved by using `-H:+PreserveFramePointer` as an image build argument.

An alternative solution is to make perf use dwarf debug info (specifically debug_frame data) to help unwind stack frames.
To make this work, the image needs to be built with `-g` (to generate debuginfo), and `perf record` needs to use the argument `--call-graph dwarf` to make sure dwarf debug info (instead of frame pointers) is used for stack unwinding.

## Related Documentation

- [Debug Native Executables with GDB](guides/debug-native-executables-with-gdb.md)

# Debugging and Diagnostics

Native Image provides utilities for debugging and inspecting the produced binary:
 - For debugging produced binaries and obtaining performance profile statistics, see [Debug Information](DebugInfo.md)
 - For generating heap dumps, see [Heap Dump Support](guides/create-heap-dump-from-native-executable.md)
 - For JFR events recording, see [JDK Flight Recorder (JFR)](JFR.md)
 - For checking which methods were included in a native executable or a shared library, use the [Inspection Tool](InspectTool.md)
 - For an overview of static analysis results, see [Static Analysis Reports](StaticAnalysisReports.md)

 # Dynamic Features of Java

When you build a native image, it only includes the reachable elements starting from your application entry point, its dependent libraries, and the JDK classes discovered through a static analysis. 
However, the reachability of some elements may not be discoverable due to Java’s dynamic features including reflection, resource access, etc. 
If an element is not reachable, it will not be included in the generated binary and this can lead to run time failures.

Thus, some dynamic Java features may require special "treatment" such as a command line option or provisioning metadata to be compatible with ahead-of-time compilation using Native Image. 

The reference information here explains how Native Image handles some dynamic features of Java:

- [Accessing Resources](Resources.md)
- [Certificate Management](CertificateManagement.md)
- [Dynamic Proxy](DynamicProxy.md)
- [Java Native Interface (JNI)](JNI.md)
- [JCA Security Services](JCASecurityServices.md)
- [Reflection](Reflection.md)
- [URL Protocols](URLProtocols.md)

# Dynamic Proxy in Native Image

Java dynamic proxies, implemented by `java.lang.reflect.Proxy`, provide a mechanism which enables object level access control by routing all method invocations through `java.lang.reflect.InvocationHandler`.
Dynamic proxy classes are generated from a list of interfaces.

Native Image does not provide machinery for generating and interpreting bytecode at run time.
Therefore all dynamic proxy classes need to be generated at image build time.

## Automatic Detection

Native Image employs a simple static analysis that detects calls to `java.lang.reflect.Proxy.newProxyInstance(ClassLoader, Class<?>[], InvocationHandler)` and `java.lang.reflect.Proxy.getProxyClass(ClassLoader, Class<?>[])`, then tries to determine the list of interfaces that define dynamic proxies automatically.
Given the list of interfaces, Native Image generates proxy classes at image build time and adds them to the native image heap.
In addition to generating the dynamic proxy class, the constructor of the generated class that takes a `java.lang.reflect.InvocationHandler` argument, i.e., the one reflectively invoked by `java.lang.reflect.Proxy.newProxyInstance(ClassLoader, Class<?>[], InvocationHandler)`, is registered for reflection so that dynamic proxy instances can be allocated at run time.

The analysis is limited to situations where the list of interfaces comes from a constant array or an array that is allocated in the same method.
For example, in the code snippets bellow the dynamic proxy interfaces can be determined automatically.

### Static Final Array:

```java
class ProxyFactory {

    private static final Class<?>[] interfaces = new Class<?>[]{java.util.Comparator.class};

    static Comparator createProxyInstanceFromConstantArray() {
        ClassLoader classLoader = ProxyFactory.class.getClassLoader();
        InvocationHandler handler = new ProxyInvocationHandler();
        return (Comparator) Proxy.newProxyInstance(classLoader, interfaces, handler);
    }
}
```

Note that the analysis operates on compiler graphs and not source code.
Therefore the following ways to declare and populate an array are equivalent from the point of view of the analysis:

```java
private static final Class<?>[] interfacesArrayPreInitialized = new Class<?>[]{java.util.Comparator.class};
```

```java
private static final Class<?>[] interfacesArrayLiteral = {java.util.Comparator.class};
```

```java
private static final Class<?>[] interfacesArrayPostInitialized = new Class<?>[1];
static {
    interfacesArrayPostInitialized[0] = java.util.Comparator.class;
}
```

However, there are no immutable arrays in Java.
Even if the array is declared as `static final`, its contents can change later on.
The simple analysis employed here does not track further changes to the array.

### New Array:

```java
class ProxyFactory {

    static Comparator createProxyInstanceFromNewArray() {
        ClassLoader classLoader = ProxyFactory.class.getClassLoader();
        InvocationHandler handler = new ProxyInvocationHandler();
        Class<?>[] interfaces = new Class<?>[]{java.util.Comparator.class};
        return (Comparator) Proxy.newProxyInstance(classLoader, interfaces, handler);
    }
}
```

Note: Just like with constant arrays, the following ways to declare and populate an array are equivalent from the point of view of the analysis:
```java
Class<?>[] interfaces = new Class<?>[]{java.util.Comparator.class};
```

```java
Class<?>[] interfaces = new Class<?>[1];
interfaces[0] = Question.class;
```

```java
Class<?>[] interfaces = {java.util.Comparator.class};
```

The static analysis covers code patterns most frequently used to define dynamic proxy classes.
For the exceptional cases where the analysis cannot discover the interface array there is also a manual dynamic proxy configuration mechanism.


### Related Documentation

- [Configure Dynamic Proxies Manually](guides/configure-dynamic-proxies.md)
- [Reachability Metadata: Dynamic Proxy](ReachabilityMetadata.md#dynamic-proxy)

# Experimental Agent Options

The `native-image-agent` tool has options which are currently experimental and might be enabled in future releases, but can also be changed or removed entirely.
These options are described here.

## Support For Predefined Classes

Native-image needs all classes to be known at image build time (a "closed-world assumption").
However, Java has support for loading new classes at runtime.
To emulate class loading, the agent can be told to trace dynamically loaded classes and save their bytecode for later use by the image builder.
This functionality can be enabled by adding `experimental-class-define-support` to the agent option string, e.g.: `-agentlib:native-image-agent=config-output-dir=config,experimental-class-define-support`
Apart from the standard configuration files, the agent will create an `agent-extracted-predefined-classes` directory in the configuration output directory and write bytecode of newly loaded classes on the go.
The configuration directory can then be used by image builder without additional tweaks,.
The classes will be loaded during the image build, but will not be initialized or made available to the application.
At runtime, if there is an attempt to load a class with the same name and bytecodes as one of the classes encountered during tracing, the predefined class will be supplied to the application.

### Known Limitations

 - Native images support "loading" a predefined class only once per execution, by just a single class loader.
 - Predefined classes are initialized when they are "loaded" at runtime and cannot be initialized at build time.
 - The agent collects all classes which are not loaded by one of the Java VM's built-in class loaders (with some exceptions), that is, from the class path or module path. This includes classes loaded by any custom class loaders.
 - Classes that are generated with varying data in their name or bytecodes, such as sequential or random numbers or timestamps, can generally not be matched to predefined classes at runtime. In these cases, the way such classes are generated needs to be adjusted.

## Printing Configuration With Origins

For debugging, it may be useful to know the origin of certain configuration entries.
By supplying `experimental-configuration-with-origins` to the agent option string, the agent will output configuration files with configuration entries broken down to the calling context (stack trace) they originate from in tree form.
This option should be used in conjunction with `config-output-dir=<path>` to tell the agent where to output the configuration files.
An example agent option string: `-agentlib:native-image-agent=config-output-dir=config-with-origins/,experimental-configuration-with-origins`

## Omitting Configuration From The Agent's Output

The agent can omit traced configuration entries present in existing configuration files.
There are two ways to specify these existing configuration files:
 - By using configuration files from the class path or module path. When `experimental-omit-config-from-classpath` is added to the agent option string, the class path and module path of the running application are scanned for `META-INF/native-image/**/*.json` configuration files.
 - By explicitly pointing the agent to an existing configuration file directory using `config-to-omit=<path>`.

## Generating Conditional Configuration Using the Agent

The agent can, using a heuristic, generate configuration with reachability conditions on user specified classes.
The agent will track configuration origins and try to deduce the conditions automatically.
User classes are specified via an agent filter file (for more information on the format, see [more about the agent](AutomaticMetadataCollection.md#caller-based-filters)).
Additionally, the resulting configuration can further be filtered using another filter file.

Currently, this feature supports two modes:
 1. Generating conditional configuration in a single run with the agent.
 2. Generating conditional configuration from multiple runs with the agent and finally merging the collected data.

### Generating Conditional Configuration During an Agent Run

To enable this mode, add `experimental-conditional-config-filter-file=<path>` to the agent's command line, where `<path>` points to an agent filter file.
Classes that are considered included by this filter will be designated as user code classes.
To further filter the generated configuration, you can use `conditional-config-class-filter-file=<path>`, where `<path>` is a path to an agent filter file.

### Generating Conditional Configuration From Multiple Agent Runs

Conditional configuration can be generated from multiple agent runs that reach different code paths in the application.
Each agent run produces configuration with metadata. `native-image-configure` is then used to merge the collected data and produce a conditional configuration.
To run the agent in this mode, add `experimental-conditional-config-part` to the agent's command line.
Once all the agent runs have finished, you can generate a conditional configuration by invoking:
```shell
native-image-configure generate-conditional --user-code-filter=<path-to-filter-file> --class-name-filter=<path-to-filter-file> --input-dir=<path-to-agent-run-output-1> --input-dir=<path-to-agent-run-ouput-2> ... --output-dir=<path-to-resulting-conditional-config>
```
where:
 - `--user-code-filter=<path-to-filter-file>`: path to an agent filter file that specifies user classes
 - (optional) `--class-name-filter=<path-to-filter-file>`: path to an agent filter file that further filters the generated config

### The Underlying Heuristics

Conditions are generated using the call tree of the application. The heuristics work as follows:
 1. For each unique method, create a list of all nodes in the call tree that correspond to the method
 2. For each unique method, if the method has more than one call node in the tree:
  - Find common configuration across all call nodes of that method
  - For each call node of the method, propagate configuration that isn't common across these calls to the caller node
 3. Repeat 2. until an iteration produced no changes in the call tree.
 4. For each node that contains configuration, generate a conditional configuration entry with the method's class as the condition.

The primary goal of this heuristic is to attempt to find where a method creates different configuration entries depending on the caller (for example, a method that wraps `Class.forName` calls.)
This implies that the heuristic will not work well for code that generates configuration through a different dependency (for example, same method returns calls `Class.forName` with different class parameters depending on a system property).
