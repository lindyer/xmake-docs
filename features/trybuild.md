
xmake v2.3.1 and above directly interface with other third-party build systems. Even if other projects do not use xmake.lua for maintenance, xmake can directly call other build tools to complete the compilation.

Then the user can directly use a third-party build tool to compile, so why use xmake to call it? The main benefits are:

1. Completely consistent behavior, simplifying the compilation process. No matter which other build system is used, you only need to execute the xmake command to compile. Users no longer need to study the different compilation processes of other tools
2. Docking the configuration environment of `xmake config`, reuse the platform detection and SDK environment detection of xmake, simplify the platform configuration
3. Docking cross-compilation environment, even for projects maintained with autotools, you can quickly cross-compile through xmake

Build systems currently supported:

* autotools (support for cross-compiling with xmake)
* xcodebuild
* cmake (support for cross-compiling with xmake)
* make
* msbuild
* scons
* meson
* bazel
* ndkbuild
* ninja

### Automatically detect build system and compile

For example, for a project maintained using cmake, executing xmake directly in the project root directory will automatically trigger a detection mechanism, detect CMakeLists.txt, and then prompt the user if cmake is needed to continue compiling.

```bash
$ xmake 
note: CMakeLists.txt found, try building it (pass -y or --confirm=y/n/d to skip confirm)?
please input: y (y/n)
-- Symbol prefix:
-- Configuring done
-- Generating done
-- Build files have been written to:/Users/ruki/Downloads/libpng-1.6.35/build
[  7%] Built target png-fix-itxt
[ 21%] Built target genfiles
[ 81%] Built target png
[ 83%] Built target png_static
...
output to/Users/ruki/Downloads/libpng-1.6.35/build/artifacts
build ok!
```

### Seamless using xmake command

Currently supports common commands such as `xmake clean`,` xmake --rebuild` and `xmake config` to seamlessly interface with third-party systems.

We can directly clean the compiled output files of the cmake maintenance project

```bash
$ xmake clean
$ xmake clean --all
```

If you bring `--all` to perform the cleanup, all files generated by autotools/cmake will be cleared, not only the object files.

The default `xmake` is docked with incremental build behavior, but we can also force a quick rebuild:

```bash
$ xmake --rebuild
```

### Manually switch the specified build system

If there are multiple build systems under maintenance in a project, such as the libpng project, which comes with autotools/cmake/makefile and other build system maintenance, xmake defaults to using autotools by default. If you want to force switch to other build systems, you can execute:

```bash
$ xmake f --trybuild=[autotools|cmake|make|msbuild|..]
$ xmake
```

In addition, the `--trybuild=` parameter is configured to manually specify the default build system, and the subsequent build process will not prompt the user for selection.

### Fastly cross compile

As we all know, although many projects maintained by autotools support cross-compilation, the configuration process of cross-compilation is very complicated. There are still many differences in different toolchain processing methods, and many pits will be stepped in the middle.

Even if you run through a toolchain's cross-compilation, if you switch to another toolchain environment, it may take a long time, and if you use xmake, you usually only need two simple commands:

!> At present cmake/autotools supports cross-compilation of xmake.

#### Cross compile android platform

```bash
$ xmake f -p android --trybuild=autotools [--ndk=xxx]
$ xmake
```

!> Among them, the --ndk parameter configuration is optional. If the user sets the ANDROID_NDK_HOME environment variable, or if the ndk is placed in ~/Library/Android/sdk/ndk-bundle, xmake can automatically detect it.

Is not it simple? If you think this is not much, then you can directly operate `./configure` to configure cross-compilation. You can see this document for comparison: [Using NDK with other compilation systems] (https://developer.android .com/ndk/guides/other_build_systems # autoconf)

To put it bluntly, you probably have to do this, you may not be able to do it once:

```bash
$ export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/$HOST_TAG
$ export AR=$TOOLCHAIN/bin/aarch64-linux-android-ar
$ export AS=$TOOLCHAIN/bin/aarch64-linux-android-as
$ export CC=$TOOLCHAIN/bin/aarch64-linux-android21-clang
$ export CXX=$TOOLCHAIN/bin/aarch64-linux-android21-clang++
$ export LD=$TOOLCHAIN/bin/aarch64-linux-android-ld
$ export RANLIB=$TOOLCHAIN/bin/aarch64-linux-android-ranlib
$ export STRIP=$TOOLCHAIN/bin/aarch64-linux-android-strip
$ ./configure --host aarch64-linux-android
$ make
```

If it is cmake, cross-compilation is not an easy task. For the android platform, this configuration is required.

```bash
$ cmake \
     -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
     -DANDROID_ABI=$ABI \
     -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
     $OTHER_ARGS
```

For the ios platform, I did not find a short-answer configuration method, but found a third-party ios toolchain configuration, which is very complicated: https://github.com/leetal/ios-cmake/blob/master/ios.toolchain.cmake

For mingw, it is another way. I have been tossing about the environment for a long time, which is very tossing.

After using xmake, whether it is cmake or autotools, cross-compilation is very simple, and the configuration method is exactly the same, streamlined and consistent.

#### Cross compile iphoneos platform

```bash
$ xmake f -p iphoneos --trybuild=[cmake|autotools]
$ xmake
```

#### Cross-compile mingw platform

```bash
$ xmake f -p mingw --trybuild=[cmake|autotools] [--mingw=xxx]
$ xmake
```

#### Using other cross-compilation toolchains

```bash
$ xmake f -p cross --trybuild=[cmake|autotools] --sdk=/xxxx
$ xmake
```

For more cross compilation configuration details, please refer to the document: [Cross Compilation](https://xmake.io/#/guide/configuration?id=cross-compilation), except for an additional `--trybuild=` parameter, all other cross-compilation configuration parameters are completely universal.

### Passing user configuration parameters

We can use `--tryconfigs=` to pass additional configuration parameters of the user to the corresponding third-party build system. For example: autotools will be passed to `. / Configure`, cmake will be passed to the` cmake` command.

```bash
$ xmake f --trybuild=autotools --tryconfigs="-enable-shared=no"
$ xmake
```

For example, the above command, pass `--enable-shared=no` to`./configure` to disable dynamic library compilation.

In addition, for `--cflags`,` --includedirs` and `--ldflags`, you don't need to pass` --tryconfigs`, you can pass the built-in parameters like `xmake config --cflags=` to pass through.

### Examples of compiling other build system 

#### General Compilation

In most cases, the compilation method after each docking system is consistent, except for the `--trybuild=` configuration parameter.

```bash
$ xmake f --trybuild=[autotools|cmake|meson|ninja|bazel|make|msbuild|xcodebuild]
$ xmake
```

!> We also need to make sure that the build tool specified by --trybuild is installed and working properly.

#### Building Android jni programs

If `jni/Android.mk` exists in the current project, then xmake can directly call ndk-build to build the jni library.

```bash
$ xmake f -p android --trybuild=ndkbuild [--ndk =]
$ xmake
```

We also provided [xmake-gradle](https://github.com/xmake-io/xmake-gradle) to build jni library in gradle, you can see [Uses xmake to build JNI in Gradle](https://xmake.io/#/plugin/more_plugins?id=gradle-plugin-jni)

