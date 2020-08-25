# Building Android native apps in CMake

Look ma, no gradle ;)

The Android NDK supports CMake as build system via a toolchain file it ships with.
However, that supports stops at generating a shared library (libwhatever.so) and it offers no help in making an actual APK that can be run, and Google expects people to just integrate their CMake scripts into their Gradle/Android Studio build system.

This is where this project comes in: it builds the [native-activity NDK sample from google](https://github.com/android/ndk-samples/tree/master/native-activity) via CMake and performs a few post build steps to package the native library created into a working APK.

Does not sound like much, but it seems there is almost zero documentation out there on how to do so - and I spent a couple of days banging my head against a wall to get this far.

Hopefully this can help some other poor soul out there...

## The good stuff

The actual packaging magic happens at the end of `CMakeLists.txt`, in `add_custom_command`

## Requirements

- Android SDK
- Android NDK
- Oracle JDK or OpenJDK
- Ninja on Windows, other platforms can use native makefiles

If you have Visual Studio (Windows or Mac) with Mobile Development workload (Xamarin) installed and install the NDK from the SDK manager you already have everything you need.
