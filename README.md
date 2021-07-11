# Training
Support code for conan trainings. `juanrh` fork

# Usage

Launch Docker contaienr for local artifactory CE and a container with the conan client already setup, and open a shell:

```bash 
cd docker_environment
docker-compose up -d
# this connects to the client container
docker exec -it conan-training bash
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
- For Cmake this implies extending the usual flow `mkdir build && cd build && cmake .. && build` by running conan install before cmake, as `mkdir build && cd build && conan install .. && cmake .. && build`, at least if we follow the convention of keeping `CMakeLists.txt` and `conanfile.txt` both at the top of our project.  
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

Conan generator just produce text files that happen to be in a format that a build system undestand. But we can even produce gcc flags directly, but leaving the `[generators]` section of `conanfile.txt` empty, and passing the `-g compiler_args` option to conan install.

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