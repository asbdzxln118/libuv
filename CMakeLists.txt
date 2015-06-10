cmake_minimum_required(VERSION 2.8)

# enable MACOSX_RPATH
if(POLICY CMP0042)
  cmake_policy(SET CMP0042 NEW)
endif()

set(LIBUV_DIR ${CMAKE_CURRENT_LIST_DIR})

include_directories(${LIBUV_DIR}/include ${LIBUV_DIR}/src)

set(UV_SRC_SHARED
    src/fs-poll.c
    src/inet.c
    src/threadpool.c
    src/uv-common.c
    src/version.c)

set(UV_SRC_POSIX
    src/unix/async.c
    src/unix/core.c
    src/unix/dl.c
    src/unix/fs.c
    src/unix/getaddrinfo.c
    src/unix/getnameinfo.c
    src/unix/loop.c
    src/unix/loop-watcher.c
    src/unix/pipe.c
    src/unix/poll.c
    src/unix/process.c
    src/unix/signal.c
    src/unix/stream.c
    src/unix/tcp.c
    src/unix/thread.c
    src/unix/timer.c
    src/unix/tty.c
    src/unix/udp.c)

set(UV_SRC_WIN
    src/win/async.c
    src/win/core.c
    src/win/dl.c
    src/win/error.c
    src/win/fs-event.c
    src/win/fs.c
    src/win/getaddrinfo.c
    src/win/getnameinfo.c
    src/win/handle.c
    src/win/loop-watcher.c
    src/win/pipe.c
    src/win/poll.c
    src/win/process-stdio.c
    src/win/process.c
    src/win/req.c
    src/win/signal.c
    src/win/stream.c
    src/win/tcp.c
    src/win/thread.c
    src/win/timer.c
    src/win/tty.c
    src/win/udp.c
    src/win/util.c
    src/win/winapi.c
    src/win/winsock.c)

if(WIN32)
    include_directories(${LIBUV_DIR}/win)
    set(UV_SOURCES ${UV_SRC_SHARED} ${UV_SRC_WIN})
else()
    set(POSIX YES)
    include_directories(${LIBUV_DIR}/unix)
    set(UV_SOURCES ${UV_SRC_SHARED} ${UV_SRC_POSIX})
endif()


# Platform specific sources
if(MAC OR IOS)
    set(UV_SOURCES ${UV_SOURCES}
        src/unix/darwin.c
        src/unix/kqueue.c
        src/unix/fsevents.c
        src/unix/darwin-proctitle.c)
endif()

if(LINUX)
    set(UV_SOURCES ${UV_SOURCES}
        src/unix/linux-core.c
        src/unix/linux-inotify.c
        src/unix/linux-syscalls.c)
endif()

if(ANDROID)
    set(UV_SOURCES ${UV_SOURCES}
        src/unix/linux-core.c
        src/unix/linux-inotify.c
        src/unix/linux-syscalls.c
        src/unix/pthread-fixes.c
        src/unix/android-ifaddrs.c)
endif()

if(NOT WIN32)
    # Shouldn't be here for BSDs
    set(UV_SOURCES ${UV_SOURCES}
        src/unix/proctitle.c)
endif()


################################################################################
# compile flags and link options
################################################################################

# If this is changed to shared, need to define BUILDING_UV_SHARED=1, -fPIC
add_library(uv STATIC ${UV_SOURCES})

if(CMAKE_BUILD_TYPE STREQUAL "Release")
    target_compile_options(uv PRIVATE
        -DNDEBUG
        -O2
        -fstrict-aliasing
        -fomit-frame-pointer
        -fdata-sections
        -ffunction-sections)
else()
    # Debug, etc
    target_compile_options(uv PRIVATE -fwrapv -Wno-strict-aliasing)
    set_target_properties(uv PROPERTIES
        XCODE_ATTRIBUTE_GCC_OPTIMIZATION_LEVEL "0")
endif()

if(LINUX AND NOT ANDROID)
    target_link_libraries(uv PRIVATE rt pthread)
endif()

if (LINUX)
    target_compile_options(uv PRIVATE
        -ansi
        -D_POSIX_C_SOURCE=200112
        -D_GNU_SOURCE)

    target_link_libraries(uv PRIVATE dl)
endif()

if (IOS OR MAC)
    target_compile_options(uv PRIVATE
        -fstrict-aliasing
        -D_DARWIN_USE_64_BIT_INODE=1
        -D_DARWIN_UNLIMITED_SELECT=1)

endif()

if(MAC)
    set_target_properties(uv PROPERTIES
        XCODE_ATTRIBUTE_ALWAYS_SEARCH_USER_PATHS "NO"
        XCODE_ATTRIBUTE_GCC_CW_ASM_SYNTAX "NO"
        XCODE_ATTRIBUTE_GCC_DYNAMIC_NO_PIC "NO"
        XCODE_ATTRIBUTE_GCC_ENABLE_CPP_EXCEPTIONS "NO"
        XCODE_ATTRIBUTE_GCC_ENABLE_CPP_RTTI "NO"              # -fno-rtti
        XCODE_ATTRIBUTE_GCC_ENABLE_PASCAL_STRINGS "NO"        # No -mpascal-strings
        XCODE_ATTRIBUTE_GCC_INLINES_ARE_PRIVATE_EXTERN "YES"
        XCODE_ATTRIBUTE_GCC_SYMBOLS_PRIVATE_EXTERN "YES"      # -fvisibility=hidden
        XCODE_ATTRIBUTE_GCC_THREADSAFE_STATICS "NO"           # -fno-threadsafe-statics
        XCODE_ATTRIBUTE_PREBINDING "NO"                       # No -Wl,-prebind
        XCODE_ATTRIBUTE_USE_HEADERMAP "NO")
endif()

if(ANDROID)
    target_compile_options(uv PRIVATE -fPIC)
endif()

if(NOT MAC)
    # Enable on all platforms except OS X. The antique gcc/clang that
    # ships with Xcode emits waaaay too many false positives.
    target_compile_options(uv PRIVATE -Wstrict-aliasing)
endif()

if(WIN32)
    target_compile_options(uv PRIVATE
        -D_WIN32_WINNT=0x0600
        -D_GNU_SOURCE)
    target_link_libraries(uv PRIVATE advapi32 iphlpapi psapi shell32 userenv
                             ws2_32)
endif()

if(NOT WIN32)
    target_compile_options(uv PRIVATE
        -D_LARGEFILE_SOURCE
        -D_FILE_OFFSET_BITS=64)
endif()

if(POSIX)
    target_compile_options(uv PRIVATE
        --std=gnu89 -pedantic -Wall -Wextra -Wno-unused-parameter)
    target_link_libraries(uv PRIVATE m)
endif()
