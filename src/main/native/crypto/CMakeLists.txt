cmake_minimum_required(VERSION 3.5)

project(crypto)
set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 98)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_C_FLAGS_RELEASE "-fdata-sections -ffunction-sections -fstack-protector -Os -flto -s")
set(CMAKE_CXX_FLAGS_RELEASE "-fdata-sections -ffunction-sections -fstack-protector -Os -flto -s")
add_definitions(-DED25519_CUSTOMHASH -DED25519_CUSTOMRANDOM)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

# Java JNI
include_directories("./jni")

# libsodium library
include_directories("./libsodium/src/libsodium/include")
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    add_custom_target(
        sodium
        WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/libsodium"
        COMMAND env CFLAGS='-mmacosx-version-min=${MACOSX_VERSION_MIN} -arch ${CMAKE_SYSTEM_PROCESSOR} -march=core2' | grep CFLAGS
        COMMAND env LDFLAGS='-mmacosx-version-min=${MACOSX_VERSION_MIN} -arch ${CMAKE_SYSTEM_PROCESSOR} -march=core2' | grep LDFLAGS
        COMMAND ./autogen.sh
        COMMAND ./configure --enable-shared=no --enable-minimal=yes --disable-pie
        COMMAND make clean check -j${nproc}
    )
else()
    add_custom_target(
        sodium
        WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}/libsodium"
        COMMAND ./autogen.sh
        COMMAND ./configure --enable-shared=no --enable-minimal=yes --disable-pie --host=${CMAKE_SYSTEM_PROCESSOR}-${TARGET_ABI}
        COMMAND make clean
        COMMAND make -j${nproc}
    )
endif()

# source files
set(SOURCE_FILES com_smartx_crypto_Native.cpp ripemd160.c ed25519-donna/ed25519.c)

# build
add_library(smartxcrypto SHARED ${SOURCE_FILES})
add_dependencies(smartxcrypto sodium)
set(SMARTXCRYPTO_TARGET_DIR ${CMAKE_CURRENT_LIST_DIR}/../../resources/native/${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR})
target_link_libraries(
    smartxcrypto
    "-L${CMAKE_CURRENT_LIST_DIR}/libsodium/src/libsodium/.libs -lsodium"
)
add_custom_command(
    TARGET smartxcrypto
    POST_BUILD
    COMMAND mkdir -p ${SMARTXCRYPTO_TARGET_DIR}
    COMMAND cp $<TARGET_FILE:smartxcrypto> ${SMARTXCRYPTO_TARGET_DIR}
)