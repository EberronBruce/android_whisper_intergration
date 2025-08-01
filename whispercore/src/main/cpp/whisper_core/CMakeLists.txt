cmake_minimum_required(VERSION 3.5) # for add_link_options and implicit target directories.
project("whisper.cpp" C CXX)
project("whisper.cpp" VERSION 1.7.6)
include(CheckIncludeFileCXX)

set(SOVERSION 1)

#set(CMAKE_WARN_DEPRECATED YES)
set(CMAKE_WARN_UNUSED_CLI YES)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT XCODE AND NOT MSVC AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Add path to modules
list(APPEND CMAKE_MODULE_PATH "cmake")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if (CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(WHISPER_STANDALONE ON)

    include(git-vars)

    # configure project version
    configure_file(${CMAKE_SOURCE_DIR}/bindings/javascript/package-tmpl.json ${CMAKE_SOURCE_DIR}/bindings/javascript/package.json @ONLY)
else()
    set(WHISPER_STANDALONE OFF)
endif()

if (EMSCRIPTEN)
    set(BUILD_SHARED_LIBS_DEFAULT OFF)

    option(WHISPER_WASM_SINGLE_FILE "whisper: embed WASM inside the generated whisper.js" ON)

    # TODO: without these, we get the following error:
    #       wasm-ld: error: --shared-memory is disallowed by whisper.cpp.o because it was not compiled with 'atomics' or 'bulk-memory' features.
    set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -pthread")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")

    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -s TOTAL_STACK=5242880")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -s TOTAL_STACK=5242880")

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-deprecated")
else()
    if (MINGW)
        set(BUILD_SHARED_LIBS_DEFAULT OFF)
    else()
        set(BUILD_SHARED_LIBS_DEFAULT ON)
    endif()
endif()

option(BUILD_SHARED_LIBS "build shared libraries" ${BUILD_SHARED_LIBS_DEFAULT})

#
# option list
#

# debug
option(WHISPER_ALL_WARNINGS           "whisper: enable all compiler warnings"                   ON)
option(WHISPER_ALL_WARNINGS_3RD_PARTY "whisper: enable all compiler warnings in 3rd party libs" OFF)

# build
option(WHISPER_FATAL_WARNINGS  "whisper: enable -Werror flag"               OFF)
option(WHISPER_USE_SYSTEM_GGML "whisper: use system-installed GGML library" OFF)

# sanitizers
option(WHISPER_SANITIZE_THREAD    "whisper: enable thread sanitizer"    OFF)
option(WHISPER_SANITIZE_ADDRESS   "whisper: enable address sanitizer"   OFF)
option(WHISPER_SANITIZE_UNDEFINED "whisper: enable undefined sanitizer" OFF)

# extra artifacts
option(WHISPER_BUILD_TESTS    "whisper: build tests"          ${WHISPER_STANDALONE})
option(WHISPER_BUILD_EXAMPLES "whisper: build examples"       ${WHISPER_STANDALONE})
option(WHISPER_BUILD_SERVER   "whisper: build server example" ${WHISPER_STANDALONE})

# 3rd party libs
option(WHISPER_CURL "whisper: use libcurl to download model from an URL" OFF)
option(WHISPER_SDL2 "whisper: support for libSDL2" OFF)

if (CMAKE_SYSTEM_NAME MATCHES "Linux")
    option(WHISPER_FFMPEG "whisper: support building and linking with ffmpeg libs (avcodec, swresample, ...)" OFF)
endif()

option(WHISPER_COREML                "whisper: enable Core ML framework"  OFF)
option(WHISPER_COREML_ALLOW_FALLBACK "whisper: allow non-CoreML fallback" OFF)
option(WHISPER_OPENVINO              "whisper: support for OpenVINO"      OFF)

# Required for relocatable CMake package
include(cmake/build-info.cmake)

# override ggml options
set(GGML_SANITIZE_THREAD    ${WHISPER_SANITIZE_THREAD})
set(GGML_SANITIZE_ADDRESS   ${WHISPER_SANITIZE_ADDRESS})
set(GGML_SANITIZE_UNDEFINED ${WHISPER_SANITIZE_UNDEFINED})
set(GGML_ALL_WARNINGS       ${WHISPER_ALL_WARNINGS})
set(GGML_FATAL_WARNINGS     ${WHISPER_FATAL_WARNINGS})

# transition helpers
function (whisper_option_depr TYPE OLD NEW)
    if (${OLD})
        message(${TYPE} "${OLD} is deprecated and will be removed in the future.\nUse ${NEW} instead\n")
        set(${NEW} ON)
    endif()
endfunction()

whisper_option_depr(FATAL_ERROR WHISPER_CUBLAS              GGML_CUDA)
whisper_option_depr(WARNING     WHISPER_CUDA                GGML_CUDA)
whisper_option_depr(WARNING     WHISPER_KOMPUTE             GGML_KOMPUTE)
whisper_option_depr(WARNING     WHISPER_METAL               GGML_METAL)
whisper_option_depr(WARNING     WHISPER_METAL_EMBED_LIBRARY GGML_METAL_EMBED_LIBRARY)
whisper_option_depr(WARNING     WHISPER_NATIVE              GGML_NATIVE)
whisper_option_depr(WARNING     WHISPER_OPENMP              GGML_OPENMP)
whisper_option_depr(WARNING     WHISPER_RPC                 GGML_RPC)
whisper_option_depr(WARNING     WHISPER_SYCL                GGML_SYCL)
whisper_option_depr(WARNING     WHISPER_SYCL_F16            GGML_SYCL_F16)
whisper_option_depr(WARNING     WHISPER_CCACHE              GGML_CCACHE)

if (GGML_CUDA AND NOT MSVC)
    #GGML_CUDA enabled, add the necessary compile options -Wno-deprecated-gpu-targets
    add_compile_options(-Wno-deprecated-gpu-targets)
endif()

#
# build the library
#

if (NOT TARGET ggml)
    if (WHISPER_USE_SYSTEM_GGML)
        find_package(ggml REQUIRED)
        if (NOT ggml_FOUND)
            message(FATAL_ERROR "System-installed GGML library not found.")
        endif()
        add_library(ggml ALIAS ggml::ggml)
    else()
        add_subdirectory(ggml)
        if(WIN32)
            # The following adds a _DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR macro and is a workaround for
            # the Windows C++ standard library which does not support constexpr mutexes.
            # From the release notes://github.com/microsoft/STL/wiki/Changelog
            #  Disable constexpr mutex constructor on Windows
            #  Fixed mutex's constructor to be constexpr. #3824 #4000 #4339
            #  Note: Programs that aren't following the documented restrictions on binary compatibility may encounter
            #  null dereferences in mutex machinery. You must follow this rule:
            #  When you mix binaries built by different supported versions of the toolset, the Redistributable version
            #  must be at least as new as the latest toolset used by any app component.
            #  You can define _DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR as an escape hatch.
            #
            # Specifically to whisper.cpp this would cause a crash when using the Java bindings.
            # resulting in a Invalid memory access error.
            target_compile_definitions(ggml-base PRIVATE _DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR)
        endif()
    endif()
    # ... otherwise assume ggml is added by a parent CMakeLists.txt
endif()
add_subdirectory(src)

#
# install
#

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(WHISPER_BUILD_NUMBER        ${BUILD_NUMBER})
set(WHISPER_BUILD_COMMIT        ${BUILD_COMMIT})
set(WHISPER_INSTALL_VERSION     ${CMAKE_PROJECT_VERSION})

set(WHISPER_INCLUDE_INSTALL_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Location of header  files")
set(WHISPER_LIB_INSTALL_DIR     ${CMAKE_INSTALL_LIBDIR}     CACHE PATH "Location of library files")
set(WHISPER_BIN_INSTALL_DIR     ${CMAKE_INSTALL_BINDIR}     CACHE PATH "Location of binary  files")

get_directory_property(WHISPER_TRANSIENT_DEFINES COMPILE_DEFINITIONS)

set_target_properties(whisper PROPERTIES PUBLIC_HEADER include/whisper.h)
install(TARGETS whisper LIBRARY PUBLIC_HEADER)

target_compile_definitions(whisper PRIVATE
    WHISPER_VERSION="${PROJECT_VERSION}"
)

configure_package_config_file(
        cmake/whisper-config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/whisper-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/whisper
    PATH_VARS
    WHISPER_INCLUDE_INSTALL_DIR
    WHISPER_LIB_INSTALL_DIR
    WHISPER_BIN_INSTALL_DIR )

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/whisper-version.cmake
    VERSION ${WHISPER_INSTALL_VERSION}
    COMPATIBILITY SameMajorVersion)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/whisper-config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/whisper-version.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/whisper)

configure_file(cmake/whisper.pc.in
        "${CMAKE_CURRENT_BINARY_DIR}/whisper.pc"
        @ONLY)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/whisper.pc"
        DESTINATION lib/pkgconfig)

#
# programs, examples and tests
#

if (WHISPER_BUILD_TESTS AND NOT CMAKE_JS_VERSION)
    include(CTest)
    add_subdirectory(tests)
endif ()

if (WHISPER_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if (MSVC)
    set(MSVC_WARNING_FLAGS
        /wd4101  # Unreferenced local variable
        /wd4005  # Macro redefinition
        /wd4065  # switch statement contains 'default' but no 'case' labels
        /wd4267  # Conversion from 'size_t' to a smaller type, possible loss of data
        /wd4244  # Conversion from one type to another type, possible loss of ata
        /wd4805  # Unsafe mix of type
        /wd4305  # Truncation from 'type1' to 'type2' (often double to float)
        /wd4996  # Function or variable may be unsafe/deprecated
    )
    function(disable_msvc_warnings target_name)
        if(TARGET ${target_name})
            target_compile_options(${target_name} PRIVATE ${MSVC_WARNING_FLAGS})
        endif()
    endfunction()

    if (WHISPER_BUILD_EXAMPLES)
        disable_msvc_warnings(whisper)
        disable_msvc_warnings(common)
        disable_msvc_warnings(common-sdl)
        disable_msvc_warnings(lsp)
        disable_msvc_warnings(wchess-core)
        disable_msvc_warnings(whisper-command)
        disable_msvc_warnings(whisper-cli)
        disable_msvc_warnings(whisper-server)
        disable_msvc_warnings(whisper-stream)
        disable_msvc_warnings(whisper-talk-llama)
        disable_msvc_warnings(whisper-bench)
        disable_msvc_warnings(quantize)
        disable_msvc_warnings(vad-speech-segments)
    endif()
endif()
