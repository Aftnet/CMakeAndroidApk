find_file(CMAKE_TOOLCHAIN_FILE android.toolchain.cmake HINTS
    "C:/Program Files (x86)/Android/android-sdk/ndk-bundle/build/cmake/"
)

find_program(CMAKE_MAKE_PROGRAM ninja HINTS
    "C:/Program Files (x86)/Microsoft Visual Studio/2019/Enterprise/Common7/IDE/CommonExtensions/Microsoft/CMake/Ninja/"
)

find_program(JARSIGNER_PATH jarsigner HINTS
    "C:/Program Files/Android/jdk/microsoft_dist_openjdk_1.8.0.25/bin"
)

function(FIND_ANDROID_BUILD_TOOLS)
    file(GLOB_RECURSE AAPT_PATHS "${ANDROID_NDK}/../build-tools/*/aapt2*")
    list(GET AAPT_PATHS -1 BUILD_TOOLS_PATH)
    get_filename_component(BUILD_TOOLS_PATH "${BUILD_TOOLS_PATH}" ABSOLUTE)
    get_filename_component(BUILD_TOOLS_PATH "${BUILD_TOOLS_PATH}" DIRECTORY)
    set(ANDROID_BUILD_TOOLS "${BUILD_TOOLS_PATH}" PARENT_SCOPE)
endfunction()

function(FIND_ANDROID_PLATFORM_PATH)
    get_filename_component(ANDROID_PLATFORM_PATH "${ANDROID_NDK}/../platforms/android-${ANDROID_PLATFORM}" ABSOLUTE)
    set(ANDROID_PLATFORM_PATH "${ANDROID_PLATFORM_PATH}" PARENT_SCOPE)
endfunction()

set(ANDROID_PLATFORM "29" CACHE STRING "")
set(ANDROID_ABI "x86_64" CACHE STRING "")

cmake_minimum_required(VERSION 3.17)
project(native-activity)

find_android_build_tools()
find_android_platform_path()

# build native_app_glue as a static lib
set(${CMAKE_C_FLAGS}, "${CMAKE_C_FLAGS}")
add_library(native_app_glue STATIC ${ANDROID_NDK}/sources/android/native_app_glue/android_native_app_glue.c)

# now build app's shared lib
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11 -Wall -Werror")

# Export ANativeActivity_onCreate(),
# Refer to: https://github.com/android-ndk/ndk/issues/381.
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -u ANativeActivity_onCreate")

add_library(${PROJECT_NAME} SHARED main.cpp)
set(PACKING_DIR_NAME packing)
set_property(TARGET ${PROJECT_NAME} PROPERTY LIBRARY_OUTPUT_DIRECTORY ${PACKING_DIR_NAME}/lib/${ANDROID_ABI})

target_include_directories(${PROJECT_NAME} PRIVATE ${ANDROID_NDK}/sources/android/native_app_glue)

# add lib dependencies
target_link_libraries(${PROJECT_NAME}
    android
    native_app_glue
    EGL
    GLESv1_CM
    log)

# supporting files
file(GLOB_RECURSE RES_FILES RELATIVE "${CMAKE_CURRENT_LIST_DIR}" "res/*" "*.xml")
foreach(FILE IN LISTS RES_FILES)
    configure_file(${FILE} ${PACKING_DIR_NAME}/${FILE} COPYONLY)
endforeach()

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${ANDROID_BUILD_TOOLS}/aapt2 ARGS compile -v --dir res -o resources.zip
    COMMAND ${ANDROID_BUILD_TOOLS}/aapt2 ARGS link -v -I "${ANDROID_PLATFORM_PATH}/android.jar" -o ${PROJECT_NAME}.apk.unaligned --manifest AndroidManifest.xml resources.zip
    COMMAND ${ANDROID_BUILD_TOOLS}/aapt ARGS add -v ${PROJECT_NAME}.apk.unaligned "lib/${ANDROID_ABI}/$<TARGET_FILE_NAME:${PROJECT_NAME}>"
    COMMAND ${JARSIGNER_PATH} ARGS -tsa http://timestamp.digicert.com -storepass password -keystore "${CMAKE_CURRENT_LIST_DIR}/androidsign.keystore" ${PROJECT_NAME}.apk.unaligned Alberto
    COMMAND ${ANDROID_BUILD_TOOLS}/zipalign ARGS 4 ${PROJECT_NAME}.apk.unaligned ${PROJECT_NAME}.apk
    WORKING_DIRECTORY ${PACKING_DIR_NAME}
)