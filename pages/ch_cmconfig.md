---
layout: default
title: Configure, Build and Use the Toolkit with CMake
nav: pages/ch_cmconfig
---


{{ page.title }}
=================================================

## Introduction

This chapter describes how to configure, build, and use the NCBI C++ Toolkit, or selected components of it using CMake.

[CMake](https://cmake.org) is an open-source, cross-platform family of tools designed to build, test and package software. It uses compiler-independent configuration files to generate native makefiles and workspaces which can be used in a variety of compiler environments.

At NCBI, we use NCBIptb – CMake wrapper, written in CMake scripting language. It adds many convenient features, facilitates handling of large source trees and simplifies CMake build target descriptions, while still allowing use of "native" CMake.


## Chapter Outline

-   [Configure build tree](#ch_cmconfig._Configure)

    -   [Use Conan to manage external packages](#ch_cmconfig._Configure_Conan)

-   [Use prebuilt Toolkit](#ch_cmconfig._Use_prebuilt)

    -   [Create new project](#ch_cmconfig._new_prebuilt)
    
    -   [Import project](#ch_cmconfig._import_prebuilt)

    -   [Conan package](#ch_cmconfig._Conan_prebuilt)

-   [NCBIptb build system](#ch_cmconfig._NCBIptb)

    -   [What is it?](#ch_cmconfig._What)

    -   [All features at a glance](#ch_cmconfig._Features)
    
    -   [Examples](#ch_cmconfig._Examples)
    
    -   [How does it work?](#ch_cmconfig._How)
    
    -   [Tree structure and variable scopes](#ch_cmconfig._Tree)

    -   [Directory entry](#ch_cmconfig._Dir)
    
    -   [Library and application targets.](#ch_cmconfig._Target)
    
    -   [Application target tests.](#ch_cmconfig._Test)
    
    -   [Custom target.](#ch_cmconfig._Custom)

-   [Testing](#ch_cmconfig._Testing)

-   [Inside NCBIptb](#ch_cmconfig._Inside)

    -   [Source tree structure](#ch_cmconfig._SrcTree)

    -   [Defaults](#ch_cmconfig._Defaults)

    -   [Project filters](#ch_cmconfig._Filters)

    -   [Extensions](#ch_cmconfig._Extensions)

    -   [External packages and requirements](#ch_cmconfig._External)


<a name="ch_cmconfig._Configure"></a>

## Configure build tree

Having checked out the source tree, run the following command in the root directory:

    On Linux or MacOS:   ./cmake-configure --help
    On Windows: cmake-configure.bat --help
    For XCode: ./cmake-configure Xcode --help

It lists available options used to generate the build tree. Several of them limit the build scope:

-   *--with-projects=*"FILE" – build projects listed in FILE. This is either a [Project List File](https://ncbi.github.io/cxx-toolkit/pages/ch_config#ch_config.Project_List_Files), or a list of subdirectories of *src* directory. Each entry is a regular expression (see [CMake documentation](https://cmake.org/cmake/help/latest/command/string.html#regex-specification) for details). List of subdirectories consists of entries separated by semicolon (hyphen in the beginning of the entry means that targets from that directory should be excluded). If "FILE" is an existing file, it should be in plain text with one such entry per line. For example:

```
    --with-projects="corelib$;serial;-serial/test"
    --with-projects="scripts/projects/ncbi_cpp.lst"
```

-   *--with-targets=*"NAMES" – lists targets to build. Again, each entry is a regular expression. For example:

```
    --with-targets="^cgi;-test"
```

-   *--with-tags=*"TAGS" – build targets with the listed tags only. Tag is a text label which may be assigned to a project. Tags are **not** treated as regular expressions. For example:

```
    --with-tags="core;-test"
```

Few options define requirements and compilation features:

-   *--with-components=*"LIST" – explicitly enable or disable components (external packages) listed in LIST. Many libraries and applications in the NCBI C++ Toolkit use external packages. It can be image manipulation, database interface, compression library. The list of available components depends on build environment. Usually, Toolkit projects either require or may use such components when they are available. If a project requires something which is not found, this project will be excluded from build. With this configuration option it is possible to disable components even when they are available - use minus sign (for example, *-BerkeleyDB*). If a component is listed here, but it is not found, configuration will treat it as error and fail. For example:

```
    --with-components="Z;-PCRE;XML"
```

-   *--with-features=*"LIST" - specify compilation features. Features are used to fine-tune build settings and enable or disable certain experimental settings. The list of available features depends on build environment. Some of available features are listed here:
    -   *BinRelease* - changes several settings, is used when building an application for public release
    -   *CfgMT* - on Windows adds build configurations which use static multithreaded runtime libraries
    -   *CfgProps* - on Windows, modifies Visual Studio solution to use custom Properties file (which defines build settings)
    -   *Coverage* - when using GCC compiler, sets code coverage flags
    -   *MaxDebug* - on Unix, adds *_GLIBCXX_DEBUG* compile definition
    -   *OpenMP* - on Unix, enables OpenMP API
    -   *SSE* - enables using SSE instruction set
    -   *StaticComponents* - instructs build system to use component's static libraries if they are available,
    -   *Symbols*  - adds debug symbols into release build,
    -   *UNICODE* - on Windows, enables using UNICODE character set.
 For example:

```
    --with-features="StaticComponents;Symbols;MaxDebug"
```


Examples of configuration commands:

    cmake-configure --with-dll --with-debug --with-projects="sra"
    cmake-configure --with-projects="misc"

Once the build tree is generated, go into build directory – for example, *CMake-GCC730-ReleaseDLL/build* or *CMake-VS2017\build*, and run *make [target]* command or open a generated solution in an IDE and build *target*.

<a name="ch_cmconfig._Configure_Conan"></a>

### Use Conan to manage external packages

[Conan](https://docs.conan.io/en/latest/) is a software package manager for C and C++ development. NCBI C++ Toolkit uses a number of third party libraries and packages. At NCBI, they are usually prebuilt and readily available in many configuration. Still, in certain scenarios, it might be beneficial to manage them using package manager. To instruct NCBIptb to use Conan, use *--with-conan* command line flag in configuration command, for example:

    cmake-configure --with-conan --with-projects="misc"

In this case, NCBIptb installs specified Conan packages first, and only after that looks for additional packages in known locations at NCBI. The list of Conan packages and their options is described in *src/build-system/cmake/conanfile.\*.txt* files. There are 3 lists – for Windows (*conanfile.MSVC.txt*), Unix (*conanfile.UNIX.txt*) and MacOS (*conanfile.XCODE.txt*).

There are two major releases of Conan - [v1.x](https://docs.conan.io/1/) and [v2.x](https://docs.conan.io/2/). The problem is that they are not fully compatible. That is, some older package recipes work with Conan1 only. For this reason, at present the Toolkit configuration supports Conan v1.x only.

The configuration process expects to find recipes and prebuilt packages in NCBI artifactory. Some of the packages exist at NCBI only - for example, [fastcgi](https://github.com/FastCGI-Archives/fcgi2), [ncbi-vdb](https://github.com/ncbi/ncbi-vdb), ncbicrypt. Outside of NCBI, one needs either establish a connection to NCBI artifactory (and specify that in Conan configuration [proxies](https://docs.conan.io/1/reference/config_files/conan.conf.html?highlight=proxies) ), or remove corresponding entries from *src/build-system/cmake/conanfile.\*.txt* files and rely on [Conan center](https://conan.io/center) only.

On Unix systems, by default, configuration installs *shared* components always (which use shared libraries). Depending on the requested build type, *Debug* or *Release* libraries are used. By specifying *--with-features="StaticComponents"*, one can request installation of *static* components.

On Windows, configuration installs *static* components in static (*--without-dll*) builds, and *shared* - in DLL (*--with-dll*) ones. Both, *Debug* and *Release* libraries are installed. Same as on Unix, by specifying *--with-features="StaticComponents"*, one can request installation of *static* components.

<a name="ch_cmconfig._Use_prebuilt"></a>

## Use prebuilt Toolkit

<a name="ch_cmconfig._new_prebuilt"></a>

### Create new project

The prebuilt Toolkit is available in several configurations. ***Note*** that this must be built using CMake – that is, it must contain CMake import target configuration files. To create a new project which uses libraries from it, use *new_cmake_project* script:

    new_cmake_project <name> <type> <builddir>

The script will create a subdirectory *name*, source subdirectories with a sample project *type* and a configuration script. Run the script, then build the project.

For example:

    new_cmake_project test app/basic $NCBI/c++.cmake.metastable
    cd test
    ./configure.sh

To get a list of available project types, run

    new_cmake_project --help

Using *new_cmake_project* script is convenience, not a requirement. You are free to choose your own style. For example, create *CMakeLists.txt* with the following contents:

    cmake_minimum_required(VERSION 3.7)
    project(test1)
    include($ENV{NCBI}/c++.cmake.stable/src/build-system/cmake/CMake.NCBItoolkit.cmake)
    add_executable(blast_demo blast_demo)
    target_link_libraries(blast_demo blastinput)

then configure and build it:

    cmake .
    make

NCBIptb style also works. Create *CMakeLists.txt*

    cmake_minimum_required(VERSION 3.7)
    project(test1)
    include($ENV{NCBI}/c++.cmake.stable/src/build-system/cmake/CMake.NCBItoolkit.cmake)
    NCBI_begin_app(blast_demo)
        NCBI_sources(blast_demo)
        NCBI_uses_toolkit_libraries(blastinput)
    NCBI_end_app()

Configure and build:

    cmake .
    make

<a name="ch_cmconfig._import_prebuilt"></a>

### Import project

In many cases, you work on your own project which is a part of the NCBI C++ toolkit tree, and you do not want to check out, update and rebuild the entire NCBI C++ tree. Instead, you just want to use headers and libraries of the pre-built Toolkit.

The shell script *import_cmake_project* will check out your project’s src and include directories from the repository and create required references to the prebuilt Toolkit. You then need to configure the build using generated *configure* script. For example:

    import_cmake_project test serial/datatool $NCBI/c++.cmake.metastable
    cd test
    ./configure.sh

It also possiible to use [*import_project*](https://ncbi.github.io/cxx-toolkit/pages/ch_getcode_svn.html#ch_getcode_svn.import_project_sh) script, but one has to be more specific about prebuilt tree:

    import_project serial $NCBI/c++.cmake.metastable/CMake-GCC730-Debug

The script configures the tree automatically, according to prebuilt directory settings (GCC730-Debug in the example above).

<a name="ch_cmconfig._Conan_prebuilt"></a>

### Conan package

NCBI C++ Toolkit is also available as Conan package. There are two packages, in fact - [ncbi-cxx-toolkit-public](https://github.com/ncbi/ncbi-cxx-toolkit-conan), and [ncbi-cxx-toolkit-core](https://bitbucket.ncbi.nlm.nih.gov/projects/CXX/repos/ncbi-cxx-toolkit-core-conan). The latter available at NCBI only. There is also *ncbi-cxx-toolkit-public* package in [Conan center](https://conan.io/center/recipes/ncbi-cxx-toolkit-public).

*Public* package relies on Conan center and uses publicly available 3rd party packages only. *Core* one adds internal packages available at NCBI only, and provides more  debugging and testing options. To find out, what is available, use the following Conan command

    conan search 'ncbi-cxx-toolkit*' -r all

For developers at NCBI, the following samples are available: [CGI sample](https://gitlab.be-md.ncbi.nlm.nih.gov/pd/cxxtk/cxx/cgi-sample) and [FCGI sample](https://gitlab.be-md.ncbi.nlm.nih.gov/pd/cxxtk/cxx/fcgi-sample).


<a name="ch_cmconfig._NCBIptb"></a>

## NCBIptb build system

<a name="ch_cmconfig._What"></a>

### What is it?

Imagine a large source tree with thousands of projects. It takes sources from several repositories and uses numerous external packages. It consists of "core" part and subtrees. Different teams work on multiple projects. To do their work, these teams assemble their own build trees, which include certain parts of core as well as their own projects. "Core" has several official releases; team projects have their own ones. There are several high frequency builds which work on different subtrees and ensure that everything stays compatible.

NCBIptb was designed to facilitate handling of such large source tree in a dynamic build environment. The purpose of NCBIptb is to extract from the source tree only requested projects. This includes analyzing and collecting build target dependencies on other targets, analyzing dependencies on external packages and excluding targets for which such dependencies cannot be satisfied, adding sources and headers, organizing them into source groups, defining precompiled header usage.  Doing this in "pure" CMake is either impossible or requires complex project descriptions. Still, all these tasks are pretty standard and can be automated. Using NCBIptb is convenience, not a requirement; it extends functionality but still allows using "native" CMake.

Probably, limiting the set of projects to build is not such a big problem for an automated build – it must build everything anyway. Still, it is a problem for an individual developer working in an Integrated Development Environment. IDEs can handle solutions with hundreds of build targets, but they can become very slow. And, it is simply not needed, it is a waste of computer resources. Developer needs a solution with only few build targets. NCBIptb can do exactly that.

Another challenge is working with prebuilt trees. Let us say, a developer needs to add features into some applications or libraries. He or she checks out part of the source tree. Some libraries are present in the local tree, others should be taken from the prebuilt one. The problem is that local libraries may have the same names as prebuilt ones. CMake will not add them into the build saying that these targets are already defined as "imported" ones. NCBIptb solves this problem by renaming such local build targets and adjusting target dependencies accordingly. Developer’s intervention is not required, everything is made automatically.

<a name="ch_cmconfig._Features"></a>

### All features at a glance

When analyzing source tree, NCBIptb does not expect much from  it, anything may be absent - subdirectories, projects, external components. NCBIptb issues a warning and it is up to developer to decide what to do with it.

When configuring the build tree, NCBIptb does the following:

-   Filters build targets by their location in the source tree, by name, and by tag, in any combination.

-   Collects all dependencies of requested build targets.

-   Excludes targets for which requirements are not met.

-   Adds header files.

-   Adds certain source and resource files, when required. This includes DLL or Windows GUI application entry points on Windows.

-   Creates source groups.

-   Defines precompiled header usage.

-   When using prebuilt trees, identifies local targets with the same names as imported ones, and renames them, adjusting dependencies. Using local targets has priority.

-   Defines tests.

In addition, there is an option of creating "composite" build targets. Such targets are composed of other targets, including their sources and requirements. In the build tree, such "hosted" targets are excluded and replaced with "composite" ones.

There are also tasks of source code generation, installation and testing. NCBIptb does not support them directly. Instead, there is a mechanism of plugin [extensions](#ch_cmconfig._Extensions) which handle them.

### Examples

To define a simple build target in CMake we use the following command:

    add_executable(hello hello.cpp)

The same operation in NCBIptb will look like this:

    NCBI_begin_app(hello)
    NCBI_sources(hello.cpp)
    NCBI_end_app()

So far, it does not look like NCBIptb makes a lot of sense.

Let us now create source groups, because it will look better in an IDE:

CMake:

    source_group("Source files" FILES hello.cpp)
    source_group("Header files" FILES hello.hpp)
    add_executable(hello hello.cpp hello.hpp)

In NCBIptb no changes are required, because source groups will be created automatically.

Now, let us say our app uses package X, which may be absent.
In CMake the project description will look like this:

    if (X_FOUND)
        source_group("Source files" FILES hello.cpp)
        source_group("Header files" FILES hello.hpp)
        add_executable(hello hello.cpp)
        target_include_directories(hello ${X_INCLUDE_DIRS})
        target_compile_definitions(hello ${X_DEFINITIONS})
        target_link_libraries(hello ${X_LIBRARIES})
    endif ()

In NCBIptb, only one line must be added:

    NCBI_begin_app(hello)
    NCBI_sources(hello.cpp)
    NCBI_requires(X)
    NCBI_end_app()

<a name="ch_cmconfig._How"></a>

### How does it work?

While in CMake "adding a build target" is final and cannot be reversed, in NCBIptb it is only a piece of information. The decision of whether to add the target or not depends on several factors and is made by the build system itself. To do so, NCBIptb scans the source tree two times (CMake does it only once). During the first pass it collects information about target dependencies and requirements, then it uses filters to select "proper" build targets and finally, during the second pass adds them into the generated solution or build tree.

Project filters include list of source tree subdirectories, list of build targets and list of build target "tags". They can be specified in any combination. "Tag" is only a label – for example *test* or *demo*, it has no meaning for the build system. Subdirectories and targets are treated here as [regular expressions](https://cmake.org/cmake/help/v3.14/command/string.html#regex-specification). Note that project filters is only a starting point. If you request building projects in directory ***A*** only, but they require projects from directory ***B***, the latter ones will be added automatically. If you request building application ***A*** only, all required libraries will also be added automatically.

<a name="ch_cmconfig._Tree"></a>

### Tree structure and variable scopes

CMake input files are named *CMakeLists.txt*. When one "adds a subdirectory" to the build, CMake looks for *CMakeLists.txt* file in this directory and processes it. Each of the directories in a source tree has its own variable bindings. Before processing the *CMakeLists.txt* file for a directory, CMake copies all currently defined variables and creates a new scope. All changes to variables are reflected in the current scope only. They are propagated to subdirectories, but not to the parent one.

*CMakeLists.txt* in turn can include other CMake files – this does not create a new scope. This is good and bad at the same time. We will return to this topic shortly.

So, in *CMakeLists.txt* one can call *add_subdirectory*, *include* a CMake file, or define a build target. In NCBIptb we use *NCBI_add_subdirectory*, *NCBI_add_library*, *NCBI_add_app* and *NCBI_add_target* instead. While NCBIptb does not prohibit defining a build target in *CMakeLists.txt*, we find it beneficial to define them in separate files. Also, NCBIptb creates a separate scope for each such file. That is, variables defined in *CMakeLists.txt* are propagated to each target definition file in the current directory and all subdirectories. At the same time, variables defined in target definition file are not propagated anywhere - they are guaranteed to be local.

What if it was not so? What if, instead of calling *NCBI_add_target()*, we simply *included* target definition file? It is a common scenario when there are several target definitions in a single directory. Once so, variables defined in one file could silently affect others and potentially create a lot of confusion. Creating a separate scope for each target definition eliminates this interdependency.

<a name="ch_cmconfig._Dir"></a>

### Directory entry

Normally, *CMakeLists.txt* contains the following function calls: *NCBI_add_subdirectory*, *NCBI_add_library*, *NCBI_add_app* and *NCBI_add_target*.

-   **NCBI_add_subdirectory**(a b) – adds subdirectories *a* and *b*. In order to be processed, a subdirectory must exist and have CMakeLists.txt file. Otherwise, a warning will be printed, and the entry skipped.

-   **NCBI_add_library**(a b) – adds library build targets. In the current directory it looks for files named *CMakeLists.a.lib.txt* and *CMakeLists.b.lib.txt*.

-   **NCBI_add_app**(a b) – adds application build targets. In the current directory it looks for files named *CMakeLists.a.app.txt* and *CMakeLists.b.app.txt*.

-   **NCBI_add_target**(a b) – adds custom targets. In the current directory it looks for files named *CMakeLists.a.txt* and *CMakeLists.b.txt*.

*CMakeLists.txt* may also contain calls to the following functions: *NCBI_headers*, *NCBI_disable_pch*, *NCBI_enable_pch*, *NCBI_requires*, *NCBI_optional_components*, *NCBI_add_definitions*, *NCBI_add_include_directories*, *NCBI_uses_toolkit_libraries*, *NCBI_uses_external_libraries*, *NCBI_project_tags*, *NCBI_project_watchers*. If so, settings defined by them will affect all targets defined in this directory and its subdirectories.

<a name="ch_cmconfig._Target"></a>

### Library and application targets

Definition of a library begins with *NCBI_begin_lib* and ends with *NCBI_end_lib*; definition of an application begins with *NCBI_begin_app* and ends with *NCBI_end_app*. Otherwise, there is not much difference between them. All calls to other NCBIptb functions must be put between these two.

-   **NCBI_begin_lib**(name)/**NCBI_begin_app**(name) – begins definition of a library or an application target called *name*.

-   **NCBI_end_lib**(result)/**NCBI_end_app**(result) – ends the definition. Optional argument *result* becomes TRUE when the target was indeed added to the build. This makes it possible to add "native" CMake commands or properties to the target:

```
    NCBI_begin_app(name)
    ...
    NCBI_end_app(result)
    if (result)
    ...
    endif()
```

-   **NCBI_sources**(list of source files) – adds source files to the target.

-   **NCBI_generated_sources**(list of source files) – adds sources which might not exists initially, but will be generated somehow during the build.

-   **NCBI_headers**(list of header file masks) – adds header files to the target. By default, NCBIptb adds all header files it can find both in the current source directory and corresponding include directory. If for some reason it is undesirable, this is the way to reduce the list.

-   **NCBI_dataspec**() – adds data specifications: ASN.1, DTD, XML schema, JSON schema, WSDL or PROTOBUF. Adding data specification implies generating C++ data classes. It also means that corresponding generated source files will be added to the target automatically by NCBIptb.

-   **NCBI_resources**(list of resource files) – adds Windows resource files to the target.

-   **NCBI_requires**(list of components) – adds requirements. If a requirement is not met, the target will be excluded from the build automatically. Note that it acts recursively through project dependencies, in the sense that if project A has "NCBI_requires(X)" and project B has "NCBI_uses_toolkit_libraries(A)" and X is not met, then project B won't be attempted to build.

-   **NCBI_optional_components**(list of components) – adds optional components. If a component is not found (or requirement is not met), NCBIptb will print a warning and the target will still be added to the build.

-   **NCBI_enable_pch**(), **NCBI_disable_pch**(), **NCBI_set_pch_header**(name), **NCBI_set_pch_define**(define), **NCBI_disable_pch_for**(list of files) – define the usage of precompiled headers. Most of the time, default behavior is enough, and these calls are not needed. Still some projects prefer to precompile their own headers, or do not want it at all.

-   **NCBI_uses_toolkit_libraries**(list of libraries) – adds dependencies on other libraries in the same build tree.

-   **NCBI_optional_toolkit_libraries**(COMPONENT list of libraries) – adds dependencies on other libraries in the same build tree depending on the availability of component *COMPONENT*.

-   **NCBI_uses_external_libraries**(list of libraries) – adds external libraries to the build target. Probably, a better way of doing this is by using *requirements* in *NCBI_requires*, but if a library should be added to one or two projects only, then this might be an easier way.

-   **NCBI_add_definitions**(list) – add compiler definitions to the target.

-   **NCBI_add_include_directories**(list) – adds include directories.

-   **NCBI_project_tags**(list) – adds tags to the target. A target may have an unlimited number of tags.

-   **NCBI_project_watchers**(list) - list of developers who will receive email notification when application test fail.


<a name="ch_cmconfig._Test"></a>

### Application target tests.

There are different approaches to testing. CMake has test infrastructure, or organization can use their own one. NCBIptb defines tests in a standard way. Then, a special module translates this definition into one used in a specific case.

All tests must be described inside application target definition, that is between *NCBI_begin_app* and *NCBI_end_app* calls.

There are two forms of test definition – short and long one.

Short one consists of a single function call – **NCBI_add_test**(command and arguments).

Long form allows to define additional requirements and add test assets.

-   **NCBI_begin_test**(name) – begins definition of a test. *name* is optional parameter, if it is absent an automatically generated name will be used.

-   **NCBI_end_test**() – ends the definition.

-   **NCBI_set_test_command**(command) – usually this is the name of the application but may be a script for example.

-   **NCBI_set_test_arguments**(arguments) – command arguments.

-   **NCBI_set_test_assets**(list of files and directories) – lists files and directories required for the test. These are relative to the current source directory. Note that the test will run in a separate directory, probably created specifically for this test. So, if it requires data files, they must be copied to that directory. 

-   **NCBI_set_test_requires**(list of components) - adds test requirements. If a requirement is not met, the test will not be added.

-   **NCBI_set_test_timeout**(seconds) – test timeout in seconds.

<a name="ch_cmconfig._Custom"></a>

### Custom target

Definition of a custom target begins with *NCBI_begin_custom_target*(name) and ends with *NCBI_end_custom_target*(result). It also requires a definition of a function which creates this target, that is calls *add_custom_target* CMake function.

The definition consists of target requirements - *NCBI_requires*, dependencies on other targets in the same build tree – *NCBI_custom_target_dependencies* and the callback function which defines the target – *NCBI_custom_target_definition*. 

That is, the definition looks as follows:

    function(xxx_definition) 
        ...
        add_custom_target(name ...)
    endfunction()
    NCBI_begin_custom_target(name)
        NCBI_requires( list of components)
        NCBI_custom_target_dependencies(list of toolkit libraries or apps)
        NCBI_custom_target_definition( xxx_definition)
    NCBI_end_custom_target(result)

This approach allows to define custom target only when all the requirements are met and collect target dependencies automatically.

<a name="ch_cmconfig._Testing"></a>

## Testing

The build system supports two test frameworks - NCBI and CMake one. To use NCBI test framework on Linux, in the root directory execute the following command:

    make check; ./check.sh run

To use CMake testing framework:

    On Linux: make test
    In Visual Studio or XCode: "build" RUN_TESTS target

Refer to [CMake documentation](https://cmake.org/cmake/help/v3.14/manual/ctest.1.html) for details.
In case of CMake testing framework, test outputs can be found in *CMake-GCC730-ReleaseDLL/testing* directory.

<a name="ch_cmconfig._Inside"></a>

## Inside NCBIptb

Normally, developer should not care about anything described in this section. NCBIptb is a part of a bigger system (**core** part, but still only a part), which defines build configurations, source tree structure, configuration files, external components and so on. All settings described here should be defined by this system.

<a name="ch_cmconfig._SrcTree"></a>

### Source tree structure

NCBIptb expects the following variables to be defined: **NCBI_SRC_ROOT** and **NCBI_INC_ROOT**. *NCBI_SRC_ROOT* is the root directory of all source files, private headers and project definitions, *NCBI_INC_ROOT* is the root directory of public headers. For example, if a project is defined in *${NCBI_SRC_ROOT}/a/b/c*, the system looks for its headers in *${NCBI_INC_ROOT}/a/b/c*.

<a name="ch_cmconfig._Defaults"></a>

### Defaults

The following values, which affect project generation, may be defined:

-   **NCBI_DEFAULT_USEPCH** - allows the use of precompiled headers (*ON/OFF*).

-   **NCBI_DEFAULT_PCH** - file name of the precompiled header.

-   **NCBI_DEFAULT_PCH_DEFINE** - additional compile definition to use with the precompiled header.

-   **NCBI_DEFAULT_HEADERS** - list of file masks (globbing expressions) to use when looking for headers. For example: *\*.h\*;\*.inl*.

-   **NCBI_DEFAULT_DLLENTRY** - Source file (with path) which should be added into all shared libraries. For example, one with the DLL entry (DllMain) definition on Windows.

-   **NCBI_DEFAULT_GUIENTRY** - Source file (with path) which should be added into all GUI applications. For example, one with the WinMain definition on Windows.

-   **NCBI_DEFAULT_RESOURCES** - Resource file (with path) which should be added into all applications.

<a name="ch_cmconfig._Filters"></a>

### Project filters

The following values affect project filtering. If none of them is defined, there is no filtering and all projects are allowed.

-   **NCBI_PTBCFG_PROJECT_LIST** - Either list of relative paths (regular expressions), or a full path to the file which contains such a list. Paths must be relative to *${NCBI_SRC_ROOT}*. For example: *corelib$;serial;-serial/test*, or */home/user/scripts/projects/ncbi_cpp.lst*

-   **NCBI_PTBCFG_PROJECT_TARGETS** - lists targets to build, each entry is a regular expression. For example: *^cgi;-test*.

-   **NCBI_PTBCFG_PROJECT_TAGS** - build targets with the listed tags only. Tags are **not** treated as regular expressions. For example: *core;-test*.

<a name="ch_cmconfig._Extensions"></a>

### Extensions

NCBIptb has a very general functionality. It analyzes source tree and selects build targets, but there is a number of other tasks. For example, source file generation, testing, installation. To make such extensions possible, NCBIptb implements a mechanism of plugins, or hooks. Hook is a function which is being called when a certain event occurs. The following events are defined:

-   **TARGET_PARSED** - a project is parsed

-   **ALL_PARSED**    - all projects in the source tree are parsed

-   **COLLECTED**     - all build targets are collected

-   **TARGET_ADDED**  - build target is added

-   **ALL_ADDED**     - all targets are added

-   **DATASPEC**      - process data specification

Here is an example of hook definition:

    function(NCBI_internal_AddCMakeTest _variable _access _value)
    ...
    endfunction()
    NCBI_register_hook(TARGET_ADDED NCBI_internal_AddCMakeTest)

The mechanism utilizes [variable_watch](https://cmake.org/cmake/help/latest/command/variable_watch.html) CMake function, which means the arguments of the callback function are defined by CMake and cannot be changed. Still, to do their job, the hooks have access to all variables defined by NCBIptb and elsewhere.

<a name="ch_cmconfig._External"></a>

### External packages and requirements

The way NCBIptb project uses external packages is by declaring

    NCBI_requires(X)
    NCBI_optional_components(Y)

X and Y may be anything, but they should be described, which means the following variables should be defined:

-   **NCBI_COMPONENT_X_FOUND** - defined as *YES* if component *X* is found

-   **NCBI_COMPONENT_X_INCLUDE** - additional include directories

-   **NCBI_COMPONENT_X_LIBS** - addtional libraries

-   **NCBI_COMPONENT_X_DEFINES** - additional compile definitions

Note that *X* should not necessarily be a package, in which case it may be defined as simple

    set(NCBI_COMPONENT_X_FOUND YES)

or, if it is indeed a package, the definition might look like this:

    find_package(X)
    if(X_FOUND)
        set(NCBI_COMPONENT_X_FOUND YES)
        set(NCBI_COMPONENT_X_INCLUDE ${X_INCLUDE_DIRS})
        set(NCBI_COMPONENT_X_LIBS ${X_LIBRARIES})
    endif()

Sure, there is a number of ways of defining them.

When analyzing the source tree, if NCBIptb sees that a project requires *X*, it will check the value of *NCBI_COMPONENT_X_FOUND*, and will either exclude the project from the build, or add appropriate build settings.

