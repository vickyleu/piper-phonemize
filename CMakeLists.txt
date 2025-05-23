cmake_minimum_required(VERSION 3.14)

set(CMAKE_VERBOSE_MAKEFILE off)

project(
    piper_phonemize
    VERSION 1.2.0
    DESCRIPTION "Phonemization library for Piper text to speech system"
    HOMEPAGE_URL "https://github.com/rhasspy/piper-phonemize"
    LANGUAGES CXX
)

set(CMAKE_CXX_FLAGS_OLD "${CMAKE_CXX_FLAGS}")
set(CMAKE_C_FLAGS_OLD "${CMAKE_C_FLAGS}")
if(MSVC)
    # Force compiler to use UTF-8 for IPA constants
    add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")

elseif(NOT APPLE)
    add_compile_options(-Wno-psabi -march=armv7-a -mfloat-abi=hard)
    # Linux flags
    add_compile_options(-Wall -Wextra)
    # C++ 文件包含 cstdint
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -include cstdint")
    if (BUILD_SHARED_LIBS)
        # 设置运行时路径
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-rpath,'$ORIGIN'")
    else ()
        set(CMAKE_STATIC_LINKER_FLAGS "")
    endif ()

endif()

if (BUILD_SHARED_LIBS)
    add_library(
            piper_phonemize SHARED
            src/phonemize.cpp
            src/phoneme_ids.cpp
            src/tashkeel.cpp
            src/shared.cpp
    )
else ()
    add_library(
            piper_phonemize STATIC
            src/phonemize.cpp
            src/phoneme_ids.cpp
            src/tashkeel.cpp
            src/shared.cpp
    )
endif ()



set_target_properties(piper_phonemize PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

# ---- onnxruntime ---

# Look for onnxruntime files in <root>/lib
if(NOT DEFINED ONNXRUNTIME_DIR)
    if(NOT DEFINED ONNXRUNTIME_VERSION)
        set(ONNXRUNTIME_VERSION "1.16.3")
    endif()

    if(WIN32)
        # Windows x86-64
        set(ONNXRUNTIME_PREFIX "onnxruntime-win-x64-${ONNXRUNTIME_VERSION}")
        set(ONNXRUNTIME_EXT "zip")
    elseif (APPLE)
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # MacOS x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-x86_64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL arm64)
            # MacOS Apple Silicon
            set(ONNXRUNTIME_PREFIX "onnxruntime-osx-arm64-${ONNXRUNTIME_VERSION}")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "tgz")
    else()
        if(CMAKE_SYSTEM_PROCESSOR STREQUAL x86_64)
            # Linux x86-64
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL aarch64)
            # Linux ARM 64-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-aarch64-${ONNXRUNTIME_VERSION}")
        elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^armv7(l)?$")
            # Linux ARM 32-bit
            set(ONNXRUNTIME_PREFIX "onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}")
            set(ONNXRUNTIME_URL "https://github.com/vickyleu/prebuilt-apps/releases/download/1.0.0/onnxruntime-linux-arm32-${ONNXRUNTIME_VERSION}.zip")
        else()
            message(FATAL_ERROR "Unsupported architecture for onnxruntime")
        endif()

        set(ONNXRUNTIME_EXT "zip")
    endif()

    if(NOT DEFINED ONNXRUNTIME_URL)
        set(ONNXRUNTIME_URL "https://github.com/microsoft/onnxruntime/releases/download/v${ONNXRUNTIME_VERSION}/${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
    endif()

    set(ONNXRUNTIME_FILENAME "${ONNXRUNTIME_PREFIX}.${ONNXRUNTIME_EXT}")
    set(ONNXRUNTIME_DIR "${CMAKE_CURRENT_LIST_DIR}/lib/${ONNXRUNTIME_PREFIX}")

    if(NOT EXISTS "${ONNXRUNTIME_DIR}")
        if(NOT EXISTS "download/${ONNXRUNTIME_FILENAME}")
            # Download onnxruntime release
            message("Downloading ${ONNXRUNTIME_URL}")
            file(DOWNLOAD "${ONNXRUNTIME_URL}" "download/${ONNXRUNTIME_FILENAME}")
        endif()

        # Extract .zip or .tgz to a directory like lib/onnxruntime-linux-x64-1.14.1/
        file(ARCHIVE_EXTRACT INPUT "download/${ONNXRUNTIME_FILENAME}" DESTINATION "${CMAKE_CURRENT_LIST_DIR}/lib")
    endif()
endif()

# ---- espeak-ng ---

if(NOT DEFINED ESPEAK_NG_DIR)
    set(ESPEAK_NG_DIR "${CMAKE_CURRENT_BINARY_DIR}/ei")
    # 获取当前工具链文件的绝对路径
    get_filename_component(TOOLCHAIN_FILE_ABS "${CMAKE_TOOLCHAIN_FILE}" ABSOLUTE)
    include(ExternalProject)
    ExternalProject_Add(
        espeak_ng_external
        PREFIX "${CMAKE_CURRENT_BINARY_DIR}/e"
#        URL "https://github.com/vickyleu/espeak-ng/archive/refs/tags/2025.4.19-5.zip"
        URL "https://github.com/vickyleu/espeak-ng/archive/refs/heads/master.zip"
        CMAKE_ARGS
            -DCMAKE_INSTALL_PREFIX:PATH=${ESPEAK_NG_DIR}
            -DCMAKE_TOOLCHAIN_FILE:FILEPATH=${TOOLCHAIN_FILE_ABS}
            -DFMT_TEST:BOOL=OFF
            -DUSE_ASYNC:BOOL=OFF
            -DBUILD_SHARED_LIBS:BOOL=OFF
            -DUSE_MBROLA:BOOL=OFF
            -DUSE_LIBSONIC:BOOL=OFF
            -DUSE_LIBPCAUDIO:BOOL=OFF
            -DBUILD_ESPEAK_NG_EXE:BOOL=OFF # missing this mother fucker macro
            -DBUILD_ESPEAK_NG_TESTS:BOOL=OFF
            -DUSE_KLATT:BOOL=OFF
            -DUSE_SPEECHPLAYER:BOOL=OFF
            -DEXTRA_cmn:BOOL=ON
            -DEXTRA_ru:BOOL=ON
            -DCMAKE_C_FLAGS="-D_FILE_OFFSET_BITS=64 ${CMAKE_C_FLAGS_OLD} "
            -DCMAKE_CXX_FLAGS="-D_FILE_OFFSET_BITS=64 ${CMAKE_CXX_FLAGS_OLD} "  # 关键隔离选项
            --no-warn-unused-cli  # 抑制无关警告
    )
    add_dependencies(piper_phonemize espeak_ng_external)
endif()


# ---- Declare library ----
set(THREADS_PREFER_PTHREAD_FLAG ON)            # 优先使用 -pthread
find_package(Threads REQUIRED)                 # 查找线程库
target_include_directories(
    piper_phonemize PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
    ${ONNXRUNTIME_DIR}/include
)

target_link_directories(
    piper_phonemize PUBLIC
    ${ESPEAK_NG_DIR}/lib
    ${ONNXRUNTIME_DIR}/lib
)

target_link_libraries(
    # 使用关键字签名，避免 plain/keyword 混用
    piper_phonemize  PRIVATE
    espeak-ng
    onnxruntime
    Threads::Threads    # 推荐用 Threads 模块链接 pthread
    dl
)

target_compile_features(piper_phonemize PUBLIC cxx_std_17)

# ---- Declare executable ----

add_executable(piper_phonemize_exe src/main.cpp src/phoneme_ids.cpp)

if(NOT WIN32)
    set_property(TARGET piper_phonemize_exe PROPERTY OUTPUT_NAME piper_phonemize)
endif()

target_compile_features(piper_phonemize_exe PUBLIC cxx_std_17)

target_include_directories(
    piper_phonemize_exe PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    piper_phonemize_exe PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

target_link_libraries(piper_phonemize_exe PUBLIC
    piper_phonemize
    espeak-ng
)

# ---- Declare test ----

include(CTest)
enable_testing()
add_executable(test_piper_phonemize src/test.cpp src/phoneme_ids.cpp)
add_test(
    NAME test_piper_phonemize
    COMMAND test_piper_phonemize "${ESPEAK_NG_DIR}/share/espeak-ng-data" "${CMAKE_SOURCE_DIR}/etc/libtashkeel_model.ort"
)

target_compile_features(test_piper_phonemize PUBLIC cxx_std_17)

target_include_directories(
    test_piper_phonemize PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
    ${ESPEAK_NG_DIR}/include
)

target_link_directories(
    test_piper_phonemize PUBLIC
    ${ESPEAK_NG_DIR}/lib
)

target_link_libraries(test_piper_phonemize PUBLIC
    piper_phonemize
    espeak-ng
)

# ---- Declare install targets ----

include(GNUInstallDirs)

install(
    TARGETS piper_phonemize
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(
    DIRECTORY ${CMAKE_SOURCE_DIR}/src/
    DESTINATION include/piper-phonemize
    FILES_MATCHING
    PATTERN "*.h"
    PATTERN "*.hpp")

install(
    TARGETS piper_phonemize_exe
    ARCHIVE DESTINATION ${CMAKE_INSTALL_BINDIR})

install(
    FILES ${CMAKE_SOURCE_DIR}/etc/libtashkeel_model.ort
    TYPE DATA)

# Dependencies
install(
    DIRECTORY ${ESPEAK_NG_DIR}/
    DESTINATION ${CMAKE_INSTALL_PREFIX})

install(
    DIRECTORY ${ONNXRUNTIME_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(
    DIRECTORY ${ONNXRUNTIME_DIR}/lib/
    DESTINATION ${CMAKE_INSTALL_LIBDIR})
