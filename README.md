
# Cross-compiling hammer with MinGW

If you've ever attempted to compile anything with cmake dependencies in
Visual Studio, you know why this guide exists.

This is a very step-by-step guide because frankly I need one. I'm using
Ubuntu 20.04, which does not provide you with prebuilt mingw dev
libraries. Luckly, compiling them with cmake is not hard.
## Step 1: Install your cross-compiler
I'm going to assume you're cross-compiling for 64-bit Windows from this point on.

    apt-get install mingw-w64
On an Ubuntu 20.04 system this creates us a handy directory for our mingw-w64 toolchain:

    ls /usr/x86_64-w64-mingw32
This is where we will install the development libraries required by hammer.
## Step 2: Clone hammer and dependencies
If you cloned [this repository](https://github.com/raintherrien/hammer-mingw) with --recursive then you already have [hammer](https://github.com/raintherrien/hammer) and its dependencies, otherwise:

    git submodule update --init --recursive
    ls external
You should see:
1. hammer
2. TODO
## Step 3: Begin compiling dependencies

Begin with the easy ones: [delaunator-c](https://github.com/raintherrien/delaunator-c) and [deadlock](https://github.com/raintherrien/deadlock). Both of these repos are submodules of hammer itself. For each:

    cd external/hammer/${PROJECT_NAME}
    mkdir build && cd build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../../../toolchain.cmake ..
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install
The last lines are the important ones. We need pass our toolchain configuration file to cmake and then pass our install prefix to make when installing the library.

Note: If you're not installing these dependencies globally, substitute a local directory for DESTDIR and sprinkle that directory throughout toolchain.cmake.

Next we'll compile and install [cglm](https://github.com/recp/cglm), a very cool math library. This one is another good cmake project that takes no effort:

    cd external/cglm
    mkdir build && cd build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../toolchain.cmake ..
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install

Next we'll compile [SDL2](https://github.com/libsdl-org/SDL). This one is a well designed cmake project and is quite easy to compile. Once again from the root of hammer-mingw:

    cd external/SDL
    mkdir build && cd build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../toolchain.cmake ..
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install
No different whatsoever. Good job, SDL2 team!

At this point I should point out, if it's not already obvious, this is a very basic guide. I'm not attempting to configure any of these projects. I'm simply building their default options for the most part.

Now it's time to compile [SDL2-image](https://github.com/libsdl-org/SDL_image). This one is pretty funky. I don't know what these guys were smoking when they wrote this. I find it's easiest to totally ignore their build process.

First we're going to build [zlib](https://zlib.net/), which is packaged with SDL2-image:

    cd external/SDL_image/external/zlib-1.2.11
    mkdir build && cd build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../../../toolchain.cmake ..
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install
Then we'll be able to build [libpng](http://www.libpng.org/pub/png/libpng.html), which is also packaged with SDL2-image:

    cd external/SDL_image/external/libpng-1.6.37
    mkdir build && cd build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../../../toolchain.cmake ..
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install
And finally SDL2-image is a weird pain in the ass. First we're going to edit their CMakeLists.txt:

    cd external/SDL_image
    mkdir build && cd build
    nano ../CMakeLists.txt
We're going to comment out lines 38 through 54, which pull in zlib and libpng in a dumb way. We're going to replace this block of code with this simple one-liner to link libpng:

    find_package(PNG REQUIRED)
Then we're gonna build normally, but without installing, which SDL2-image does not support:

    cmake -DCMAKE_TOOLCHAIN_FILE=../../../toolchain.cmake -DSUPPORT_JPG=0 ..
    make -j
You'll notice we build **without** JPG support. Simply because hammer doesn't require it.
Now that SDL2-image is build we're going to manually install it:

    sudo cp libSDL2_image.dll /usr/x86_64-w64-mingw32/usr/local/bin/
    sudo cp libSDL2_image.dll.a /usr/x86_64-w64-mingw32/usr/local/lib/
    sudo cp ../SDL_image.h /usr/x86_64-w64-mingw32/usr/local/include/SDL2/
Yay! This is fun, isn't it? Why even use cmake?

And finally, [GLEW](http://glew.sourceforge.net/) is super funky because it uses scripts to generate source code. But it's pretty well [documented](http://glew.sourceforge.net/advanced.html). Source code generation requires quite a few packages, including GNU make, wget, perl, and python. Alternatively GLEW provides source code in ZIP or TGZ format. Let's begin...

    cd external/glew/auto
    make # This will take forever because it downloads the OpenGL specification
Once you've either generated or downloaded GLEW's source code, compilation and install is copy-paste, with a bit of a strange directory structure:

    cd external/glew/build
    cmake -DCMAKE_TOOLCHAIN_FILE=../../../toolchain.cmake ./cmake
    sudo make DESTDIR=/usr/x86_64-w64-mingw32/ -j install

