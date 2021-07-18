# Training
Support code for conan trainings. `juanrh` fork

# Usage

Launch Docker contaienr for local artifactory CE and a container with the conan client already setup, and open a shell:

```bash 
cd docker_environment
docker-compose up -d
# this connects to the client container
docker exec -it conan-training bash

# to stop later
docker-compose stop
```

with:

- Default URL http://localhost:8082/
- [Default credentials](https://www.jfrog.com/confluence/display/RT12/FAQs#:~:text=Artifactory%20comes%20with%20a%20pre-configured%20default%20%22admin%22%20account)

As seen in training/docker_environment/conan-training/Dockerfile, the client container has the basic C++ build commands installed, plus the conan CLI with a minimal configuratio. 

# Training notes

## Conan Arquitecture

Similar to other package managerd like Maven. 3 types of repositories:

- Local repo: to the host, share code between projects of the same host
- Remote repo = Artifactory: share packages accross developers of the same org. Note Artifactory CE is open source and can run [self-hosted](https://www.jfrog.com/confluence/display/JFROG/JFrog+Self-Hosted), also there is a [cloud offering](https://jfrog.com/pricing/)
- Centralized moderated public repo: = [Conan Center](https://conan.io/center/)
  - Host public projects.
    - https://github.com/conan-io/conan-center-index contains the conan recipes to build these projects
    - Binary build service is a CI service from the conan project that runs the recipes to build new binaries on source code changes: 100+ binaries for each package version, due to different OS, hardware configs, C libraries, etc
    - The resulting binaries are uploaded to Conan Center

## Conan package model

Different to other package managers

- __Packages__ are composed of recipes. A package corresponds to a library or executable
- A __recipe__ contains several packages binaries. __Recipe references__ are of the shape `pkg/0.1@user/channel`
- Each __package binary__ is the output of the librar or executable for the package. E.g. a specific platform, debug with symbols vs optimzied, compiler etc. This simplifies multiplatform development
  - each binary corresponds to a pair __options__-__settings__. The options-setting pair is hashed into a unique id called __package id__ that identifies the package binary
  - example setting: architecture, compiler, debug/release
  - example options: shared=true/false


## Consuming Conan Packages

See basic example in `training/consumer`.  
Dependencies are tracked in 3 places: 

- conanfile.txt: this does 3 things
  1. Declare which dependencies to download. Conan downloads the libraries to `~/.conan/data`, e.g. `~/.conan/data/boost/1.72.0/_/_/package/cfef62e446e1bb07894de4523f2649afdfa94524/lib` where "cfef62e446e1bb07894de4523f2649afdfa94524" is a package id
  2. Declare the conan __generator__, which generates configuration for your build system so it is able to access the downloaded libraries in `~/.conan`. E.g. the Cmake generator generates a file `conanbuildinfo.cmake` that 
  3. Options: these affect the output of the generator, e.g. use the dependencies with dynamic or static linking
- cmake: this is your build system. It uses conventions to hook with Conan and access the downloaded dependencies. E.g. basic example:

    ```cmake
    # Using the "cmake" generator
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup(TARGETS)

    add_executable(timer timer.cpp)
    target_link_libraries(timer CONAN_PKG::poco
                                CONAN_PKG::boost)
    ```

- includes in your code: standard C++ includes, as usual when using CMake or other build system


__How does it work__: we call `conan install PATH_TO_conanfile.txt` _before_ calling our build system to make conan download our dependencies --and all their transitive dependencies-- and generate a file for our build system to access those dependencies.  

- The `conan install` command first look for the dependencies in the local cache (`~/.conan`), and download all missing dependencies from the configured remote repositories, by default Conan Center is the only remote. You can configure additinal remotes, that have a priority number (smaller implies higher priority), so conan downloads each package from the remote with higher priority that has it.
- For Cmake this implies extending the usual flow `mkdir build && cd build && cmake .. && make` by running conan install before cmake, as `mkdir build && cd build && conan install .. && cmake .. && make`, at least if we follow the convention of keeping `CMakeLists.txt` and `conanfile.txt` both at the top of our project.  
  So __Cmake doesn't really know about conan__. 
  
Example: 

```bash
juanrh@juanySlimProX15:~/git/conan/training/docker_environment$ docker exec -it conan-training bash

cd consumer
# typical Cmake workflow
mkdir build && pushd build
# but here we 1) download the conan dependencies; 2) generate
# a Cmake file to make these dependencias available to CMake
# By convention that cmake file is conanbuildinfo.cmake, and that is
# what we have included in our CMakeLists.txt
# Note conanfile.txt is in ..
conan install ..
# Now we can use CMake as usual, because now CMake can use the new file 
# conanbuildinfo.cmake to access the dependencies
cmake .. -DCMAKE_BUILD_TYPE=Release
# Or `cmake --build .` if you like it fancy: this is just a standard
# Cmake workflow at this point
make
bin/timer
```

Other basic conan commands:

```bash
# list package installed in the local cache
conan search
# search in the local cache for packages matching "zlib"
conan search zlib
# detailed info about all package binaries for that specific
# conan recipe, that are available in the local cache: e.g. shows
# debug and release versions
conan search zlib/1.2.11@
# same as above but with pretty print: only works for searches with
# recipe references
conan search zlib/1.2.11@ --table results.html

# We can search on a remote passing the `-r` option. Search is quite
# bad here, e.g. `conan search aws -r conan-center` returns nothing
# but `conan search aws-sdk-cpp -r conan-center` returns a match. Use
# the search in https://conan.io/center/ for exploration
conan search qt -r conan-center

# This shows the dependency closure for our package. THe --graph options
# generates an html with the dependency graph
conan info ..
conan info .. --graph deps.html

# install dependecies but overriding the setting to download debug 
# dependencies
conan install .. -s build_type=Debug
cmake .. -DCMAKE_BUILD_TYPE=Debug
```

Conan [generator](https://docs.conan.io/en/latest/reference/generators.html) just produce text files that happen to be in a format that a build system undestand. But we can even produce gcc flags directly, but leaving the `[generators]` section of `conanfile.txt` empty, and passing the `-g compiler_args` option to conan install.

```bash
cd consumer_gcc/
# this produces a file conanbuildinfo.args with gcc args
conan install . -g compiler_args
# gcc can [read arguments from a file](https://gcc.gnu.org/onlinedocs/gcc-4.6.3/gcc/Overall-Options.html#Overall-Options)
g++ timer.cpp @conanbuildinfo.args -o timer -std=c++11
./timer
```

This is interesting to understand how conan works, and potentially for debugging issues, but doesn't look to practical for daily development.

There are many generators. For Cmake, besides the `cmake` generator there is also the `cmake_find_package` that generates a Cmake file complying with the FindPackage covention, to avoid using Conan specific variables in our CMake, and instead adding the output forlder for the generator to the cmake module and prefix path:

```cmake
# Using the "cmake_find_package" generator
set(CMAKE_MODULE_PATH ${CMAKE_BINARY_DIR} ${CMAKE_MODULE_PATH})
set(CMAKE_PREFIX_PATH ${CMAKE_BINARY_DIR} ${CMAKE_PREFIX_PATH})
```

In this case `conan install ..` doesn't generate a single file `conanbuildinfo.cmake` like with the "cmake" generator, but several files FindPoco.cmake, FindBoost.cmake, etc, for each of the transitive dependencies


## Creating Conan Packages

Here our goal is to package a C++ library or executable into a Conan package. Basic outline of that process:

- Create a conanfile.py that defines the packaging process
  - `conan new` is a scaffolding tool that creates a template for a new project. Pass the option `-t` to create a "test package", that is kind of a unit test but to check packagin works fine
  - `conan create` builds the package and publish it locally (to ~/.conan), so it is visible by `conan search`
    - NOTE `conan create` only modifies the local conan cache, it does NOT publish to remote artifactory repos. That requires an additional step. This is very nice because it allows to freely experiment locally 
- The conanfile defines different binaries for different configuration (Release/Debug) of the package

The `conanfile.py` defines a class extending `ConanFile` that:

- Defines methods 
  - `source` to fetch the source code: typically downloads it from github, using `self.run` to run a shell command, e.g. `self.run(git clone https://githum.com/...`
  - `build` to build the package. For supported build systems we can use [build helpers aka "helper classes"](https://docs.conan.io/en/latest/reference/build_helpers.html) that are classes with methods that simplify using the existing build configuration (e.g. CMakeLists.txt) for the package we are building: e.g. run the build, parse the config files for the build to get metadata, pass options to the build, ... What this method is doing is basically __translate__ the build from the build system format to a format conan understands. Here we can also use `self.run` if we nee it
  - `package` defines how to copy the artifacts (headers, binaries like .so or .a file) from the build directory to the package folder in the conan cache (~/.conan), a process known as _"capturing artifacts"_. `self.copy` can be used for this, using globs and relative paths (to the build directory for sources and to the package directory for destinatios). Note this should support artifacts for all OSs: this works because we use globs, so e.g. in a build for Linux we don' thave dylib but we have .so
  - `package_info` defines the variables that are passed to consumers of this package, by adding them to the pre-created dictionary `self.cpp_info`: e.g. `self.cpp_info.libs = ["hello"]`, so this is not a dict but some kind of wrapper class. See more [here](https://docs.conan.io/en/latest/creating_packages/package_information.html). I guess conan generators use this to be able to translate the build to different build systems


Regarding recipe references, we saw thay have the format `pkg/0.1@user/channel`. For official ConanCenter packages we omit `user/channel`, but we need them for custom packages:

- user is usually the of a company, or a team or project
- channel by convention usually is either "testing" or "stable"

Example package creation

```bash
cd create
# This creates scaffolding conanfile.py for building https://github.com/conan-io/hello.git with CMake
# Note CMakeLists.txt is already created in that repo
$ conan new hello/0.1
File saved: conanfile.py
# Build the package in . and publish it to the local conan cache swith user=user and channel=testing 
conan create . user/testing
# we can find the new package in the conan local cache: `version = "0.1"` is specified in conanfile.py
conan@b8d324c1eea6:~/training/create$ conan search hello
Existing package recipes:

hello/0.1@user/testing
conan@b8d324c1eea6:~/training/create$ conan search hello/0.1@user/testing
Existing packages for recipe hello/0.1@user/testing:

    Package_ID: 66c5327ebdcecae0a01a863939964495fa019a06
        [options]
            fPIC: True
            shared: False
        [settings]
            arch: x86_64
            build_type: Release
            compiler: gcc
            compiler.libcxx: libstdc++11
            compiler.version: 7
            os: Linux
        Outdated from recipe: False

conan@b8d324c1eea6:~/training/create$ 
# Now create the same package but with Debug settings: note
# this is smart enough to avoid cloning the package twice
$ conan create . user/testing -s build_type=Debug
Exporting package recipe
hello/0.1@user/testing: The stored package has not changed
...
# we now see 2 binaries with 2 package ids for the same recipe reference "hello/0.1@user/testing",
# one with debug symbols and another with release profile
conan@b8d324c1eea6:~/training/create$ conan search hello/0.1@user/testing
Existing packages for recipe hello/0.1@user/testing:

    Package_ID: 1a651c5b4129ad794b88522bece2281a7453aee4
        [options]
            fPIC: True
            shared: False
        [settings]
            arch: x86_64
            build_type: Debug
            compiler: gcc
            compiler.libcxx: libstdc++11
            compiler.version: 7
            os: Linux
        Outdated from recipe: False

    Package_ID: 66c5327ebdcecae0a01a863939964495fa019a06
        [options]
            fPIC: True
            shared: False
        [settings]
            arch: x86_64
            build_type: Release
            compiler: gcc
            compiler.libcxx: libstdc++11
            compiler.version: 7
            os: Linux
        Outdated from recipe: False

conan@b8d324c1eea6:~/training/create$
```

We can now consume this package from the same host with the following simple modifications to the consumer project

```diff
diff --git a/consumer/CMakeLists.txt b/consumer/CMakeLists.txt
index 639f042..cdaf044 100644
--- a/consumer/CMakeLists.txt
+++ b/consumer/CMakeLists.txt
@@ -8,4 +8,5 @@ conan_basic_setup(TARGETS)
 
 add_executable(timer timer.cpp)
 target_link_libraries(timer CONAN_PKG::poco
-                            CONAN_PKG::boost)
+                            CONAN_PKG::boost
+                            CONAN_PKG::hello)
diff --git a/consumer/conanfile.txt b/consumer/conanfile.txt
index ace69a3..8d69c02 100644
--- a/consumer/conanfile.txt
+++ b/consumer/conanfile.txt
@@ -1,6 +1,7 @@
 [requires]
 boost/1.72.0
 poco/1.9.4
+hello/0.1@user/testing^M
 
 [generators]
 cmake
diff --git a/consumer/timer.cpp b/consumer/timer.cpp
index a53216c..fb2e255 100644
--- a/consumer/timer.cpp
+++ b/consumer/timer.cpp
@@ -2,6 +2,7 @@
 #include <Poco/Thread.h>
 #include <Poco/Stopwatch.h>
 #include <boost/regex.hpp>
+#include "hello.h"
 
 #include <string>
 #include <iostream>
@@ -22,6 +23,8 @@ private:
 };
 
 int main(int argc, char** argv){
+    hello();
+
     TimerExample example;
     Timer timer(250, 500);
     timer.start(TimerCallback<TimerExample>(example, &TimerExample::onTimer));
```


We can check the local package cache for the artifacts but also for the build directory used, e.g. `~/.conan/data/hello/0.1/user/testing/build/` for the example above, with subdirectories for each package id. Id needed we can delete the cache with `conan remove "*" -f`, which deletes the whole cache, or using a more restrictive glob pattern to delete specific packages  

### Testing packages

Regarding __test package__, if we passed `-t` to `conan new` that creates a `test_pacakge` directory for the test package, that has its own conanfile.py but that 1) implicitly depends on the test subject package; 2) doesn't define `source`, but it has to define `build` and also `test` that defines the test (usually calling a shell command called with `self.run`); 3) doesn't define `package` or `package_info`, because they have no consumers. 

### Packaging local projects, and fetching code from other sources

A conan recipe that packages github projects is called __out-of-source recipe__. If you want to package private projects you can use an __in source recipe__, that uses source code in the same folder as the conan recipe.  
For that pass `-s` to `conan new`. That creates scaffolding for a simple C++ project. Also, in `conanfile.py` we'll have `exports_sources = "src/*"` as a class member that replaces the `source` method. 

We can also use the `scm` class method to fetch the code from supported source control systems like git or SVN. This is just a shortcut for defining a `source` method. It's not clear to me which convention conan uses to make the files fetched by the `source` method accessible to other methods. I assume `conan create` runs in some working directory, and it just assumes the files are downloaded there, and that is how `self.run("git clone ...` for `source` works fine. I guess `exports_sources = "src/*"` basically leads to synthesizing a `source` method that does a `cp` from `CONANFILE_ROOT_ABS_PATH/exports_sources` to the current working directory. That way `exports_sources` would be just another convenient alias

## Sharing conan packages

We can use the public Conan Center, or our own artifactory instance, they are basically the same software.  

The artifactory repos are called __remotes__, and are listed with `conan remote list`. 

- I create a local conan repo called "myconanrepo" in the artifactory instance running in a container configured at the top of this doc. I can add it as a remote called "artifactory" as follows: `conan remote add artifactory http://jfrog-artifactory-training:8081/artifactory/api/conan/myconanrepo`
- Now I see the new remote:

  ```bash
  conan@b8d324c1eea6:~/training$ conan remote list
  conan-center: https://conan.bintray.com [Verify SSL: True]
  artifactory: http://jfrog-artifactory-training:8081/artifactory/api/conan/myconanrepo [Verify SSL: True]
  conan@b8d324c1eea6:~/training$
  ``` 

- I can use the new remote passing `-r` to the `conan` command, which by default uses conan-center otherwise. E.g. to upload packages:

  ```bash
  # upload a recipe
  conan upload "hello" -r artifactory
  # upload a recipe and binaries
  conan upload "hello" -r artifactory --all --confirm

  # on a consumer package, install from a remote
  conan install .. -r=artifactory
  ```

## Build configuration && cross-build

Build configuration can be customized using both __settings__ and __options__, which are both hashed to build the package id.

- __options__: e.g. shared vs static libs
  - Options __apply to a single package__ and are defined in the recipe 
    - defined as a class member `options` of the class that extends `ConanFile`, the value should be a dictionary from key to the list of possible values for the key. Also a class member `default_options` define the default values, as a key-value dict
    - set on the call to `conan create` passing `-o key:value` for `key` the value to use 
- __settings__: e.g. build type (Debug/release), compiler options (optimizations, thread support, etc), compiler itself (e.g. gcc, clang, ...), cross compilation config (which implies compiler etc)
  - Settings __apply to the builds of all packages in the dependency tree__ aka dependency closure
    - defined in `~/.conan/settings.yml`

so __the difference between a setting and an option is the scope__

I understand that when we use build helpers like [CMake](https://docs.conan.io/en/latest/reference/build_helpers/cmake.html), calling them on the `buid` method as e.g. `cmake.configure(source_folder="hello")` then those build helper classes access the settings and options to look for standard key like build type, and adjust the build accordingly.  

- This implies that __if we define the`build` method manually then we have to manually comply with the settings and options__.  
  - E.g. the options are available in `self.options` with an attribute for each options key. 
  - If we are adding a custom option We can use features like `cmake.definitions` before the calls to `cmake.configure` and `cmake.build`, to change the build behaviour  

The commands `conan inspect` and `conan get` can help debugging configuration issues

Conan __profiles__ are a way to reuse a set of settings, options and also env vars by adding the to a profile file. We always use a profile, the default profile "default" is defined by conan during installation by inspecting the system

- pass `-pr=PROFILE` to use a profile, if not specified that is equivalent to `-pr=default` 
- profile files are defined in `~/.conan/profiles`, or as a file path relative to the current directory. Profiles in `~/.conan/profiles` or in the current directory are referred by file name, other profiles are referred as a path relative to the current directory
- Use `conan config install` helper to easily edit a profile in `~/.conan/profiles`, or download it there from a remote source
- profile files have an `[env]` section to define env vars used during the build, e.g. `CC` and `CXX`
- profile files have some _modularity features_. E.g. in this one:

  ```
  CROSS_GCC=arm-linux-gnueabihf

  include(default)

  [setting]
  arch=armv7

  [env]
  CC=$CROSS_GCC-gcc
  CXX=$CROSS_GCC-g++
  ```

  we can see

  - Usage of `include` to reuse the default profile
  - Usage of `$CROSS_GCC` to expand the variable `CROSS_GCC` defined above

- We can also specify _specific setting and env vars for specific packages_: e.g. adding `OpenSSL:compiler.version=4.8` to have that `compiler.version` setting only affecting the "OpenSSL" package
- We can even pass several profiles to `conan` by using `-pr` several times in the same command. Conan will mix those profiles dynamically. We can also pass individual settings and options in the CLI beside profiles, using `-s` and `-o`
- We can also add `build_requires` to as dependency of a conan package, without modifying the conan recipe. This can also be interested to install executables. See [doc on profiles](https://docs.conan.io/en/latest/reference/profiles.html) for more 
- Profiles are useful for using Conan in CI settings. 

### Example: using profiles for cross compilation to RPI

See full example in `cross_build`.

```bash
conan@b8d324c1eea6:~/training/cross_build$ cat rpi_armv7 
[settings]
os=Linux
compiler=gcc
compiler.version=7
compiler.libcxx=libstdc++11
build_type=Release
arch=armv7
os_build=Linux
arch_build=x86_64

[env]
CC=arm-linux-gnueabihf-gcc
CXX=arm-linux-gnueabihf-g++
conan@b8d324c1eea6:~/training/cross_build$ 
```

Note the usage of [conan nomenclature for cross compilation](https://docs.conan.io/en/latest/systems_cross_building/cross_building.html) where we have a __build platform__ that runs the build (e.g. your laptop, github actions server), and a __host platform__ that runs the artifact binaries (e.g. a RPI) ---when compilng a cross compiler then we also have a __target platform__ which is the platform that will run the cross compiler, which can be still be different to the host (e.g. build in Windows 10 as build platform a cross compiler to run on Ubuntu 18.04 as target platform a cross compiler that generates binaries to run on Raspbian X on RPI 3B+ as host platform). Here `arch` is the host platform architecture, and `arch_build` is the build platform architecture, and similarly for `os`.  
We can use that profile to cross compile to RPI as follows:

```bash
conan@b8d324c1eea6:~/training/cross_build$ conan create . user/testing -pr=rpi_armv7
...

-- Build files have been written to: /home/conan/training/cross_build/test_package/build/12e9898eff79f5a720b503ea00bf9e942956e02d
Scanning dependencies of target example
[ 50%] Building CXX object CMakeFiles/example.dir/example.cpp.o
[100%] Linking CXX executable bin/example
[100%] Built target example
hello/0.1@user/testing (test package): Running test()
conan@b8d324c1eea6:~/training/cross_build$ 

# We can now check this has been actually cross compiled for ARM: `file` shows the expected architecture, and we
# cannot run the binary from the build plaform as expected
conan@b8d324c1eea6:~/training/cross_build$ file /home/conan/training/cross_build/test_package/build/12e9898eff79f5a720b503ea00bf9e942956e02d/bin/example 
/home/conan/training/cross_build/test_package/build/12e9898eff79f5a720b503ea00bf9e942956e02d/bin/example: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=011b1755da62efbf94ed04be47dd5aaf6ab22bce, not stripped
conan@b8d324c1eea6:~/training/cross_build$ 

conan@b8d324c1eea6:~/training/cross_build$ /home/conan/training/cross_build/test_package/build/12e9898eff79f5a720b503ea00bf9e942956e02d/bin/example
bash: /home/conan/training/cross_build/test_package/build/12e9898eff79f5a720b503ea00bf9e942956e02d/bin/example: cannot execute binary file: Exec format error
conan@b8d324c1eea6:~/training/cross_build$ 
```

## Other Conan integrations

- [meta-conan](https://github.com/conan-io/meta-conan) is a Yocto layer to "write simple Bitbake recipes to retrieve and deploy Conan packages from an Artifactory repository.
- [conan snap integration](https://docs.conan.io/en/1.38/integrations/deployment/snap.html)

## Conclusions

- Conan is the missing sensible way of handling C++ dependencies
- To consume packages, we have generators for several build systems
  - For Cmake conan adds a single additional step to the usual flow
  - Some build systems like Cmake or Visual Studio are very well supported. Others like Meson are supported through intermediate formats like [pkg_config files](https://docs.conan.io/en/latest/reference/generators/pkg_config.html)
- To create packages, Conan doesn't simplify defining the build file by abstracting build systems, you start from a package that builds.
- Conan __helps with cross compilation__: see usage of `conan create` with a profile for cross compilation to RPI in "Example: using profiles for cross compilation to RPI" section below, which publishes RPI binaries to the local conan cache
- Conan packages can be used to distribute executables for specific platforms, it could be feasible for simple systems. We could even use `build_requires` in a profile for a meta package that collects related tooling.
  - E.g. for cross compilation we could have a fast devel flow based on a private artifactory service, from the build platform doing create and upload, and from the host platform doing install. This not be a replacement of a proper install, but could be used as an intermediate phase for quick manual testing and experimentation.
  - TBD how far we can get with this, probably something like snaps or flatpak is a more scalable option 
- Per [`system_requirements()` section in Conan docs](https://docs.conan.io/en/1.36/reference/conanfile/methods.html#system-requirements) it looks like conan sometimes calls the OS package manager to install required dependencies. This calls for using Docker to ensure the build is repetible. 
  - __TODO__ see [How to use Docker to create and cross-build C and C++ Conan packages](https://docs.conan.io/en/latest/howtos/run_conan_in_docker.html#docker-conan) to setup a better development flow. Note in fact the __docker container used in this course is a starting point__
    - Note the subtleties involved when using shared libraries. Combining Docker with [conan snap integration](https://docs.conan.io/en/1.38/integrations/deployment/snap.html) might be worth exploring
  - TODO: also check disabling system requires automatic installation in the conan.conf. see https://docs.conan.io/en/latest/reference/config_files/conan.conf.html
  
        ```
        [general]
        sysrequires_mode = disabled
        ```
