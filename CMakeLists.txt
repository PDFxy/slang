# slang - cmake entry point
cmake_minimum_required(VERSION 3.12)
project(slang VERSION 0.6.0 LANGUAGES CXX)

option(SLANG_MSVC_W4 "Enable /W4 for MSVC builds" ON)
option(SLANG_MSVC_STATIC_RUNTIME "Enable linking to the MSVC static runtime" ON)
option(SLANG_COVERAGE "Enable code coverage" OFF)
option(CI_BUILD "Enable longer running tests for CI builds" OFF)
option(FUZZ_TARGET "Enables changes to make binaries easier to fuzz test" OFF)
option(SLANG_INCLUDE_TESTS "Include test targets in the build" ON)
option(SLANG_INCLUDE_DOCS "Include documentation targets in the build" OFF)
option(SLANG_INCLUDE_LLVM "Include LLVM in the build for code generation" OFF)
option(STATIC_BUILD "Make the linked binaries static" OFF)
option(BUILD_SHARED_LIBS "Generate a shared library instead of static" OFF)
set(DOXYGENPY_PATH "" CACHE STRING "When building docs, the path to doxygen.py tool")

set(SLANG_SANITIZERS "" CACHE STRING "List of Clang sanitizers to include in build")

set(SLANG_VERSION_MAJOR 0)
set(SLANG_VERSION_MINOR 6)

# Default build type if none is set
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING
        "Build type (Release/Debug/RelWithDebInfo/MinSizeRel)" FORCE)
endif()

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" ${CMAKE_MODULE_PATH})

find_package (Python COMPONENTS Interpreter)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELEASE ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_DEBUG ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})

# Defaults for a bunch of Windows-specific junk
if(CMAKE_SYSTEM_NAME MATCHES "Windows")
    add_definitions(/DNOMINMAX)
    add_definitions(/DUNICODE)
    add_definitions(/D_UNICODE)
    add_definitions(/DWIN32_LEAN_AND_MEAN)
    add_definitions(/DNTDDI_VERSION=0x06010000)
    add_definitions(/D_WIN32_WINNT=0x0601)
    add_definitions(/D_SCL_SECURE_NO_WARNINGS)
    add_definitions(/D_CRT_SECURE_NO_WARNINGS)
    add_definitions(/D_CRT_SECURE_NO_DEPRECATE)
endif()

# Always require C++17 or later, no extensions
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # Prefer the latest standard, and enable full conformance
    add_compile_options(/std:c++17)
    add_compile_options(/permissive-)
    add_compile_options(/Zc:inline)
    add_compile_options(/Gy)        # Function-level linking
    add_compile_options(/D_SILENCE_CXX17_ITERATOR_BASE_CLASS_DEPRECATION_WARNING)

    # Ignore warnings in external headers
    add_compile_options(/experimental:external /external:anglebrackets /external:W0)

    if(CMAKE_BUILD_TYPE MATCHES "Debug")
        # Use fast linking
        string(REGEX REPLACE "/debug" "/DEBUG:FASTLINK" CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
        string(REGEX REPLACE "/debug" "/DEBUG:FASTLINK" CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS_DEBUG}")
        add_compile_options(/DDEBUG)

        # Ignore annoying "missing pdb" warnings
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /ignore:4099")
    else()
        add_compile_options(/GS-)   # Disable buffer overrun checks
        add_compile_options(/GL)    # Whole program optimization
        add_compile_options(/Zi)    # Generate debug info even in release

        set(REL_LINK_FLAGS "/LTCG /DEBUG:FULL /OPT:REF /OPT:ICF /INCREMENTAL:NO")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${REL_LINK_FLAGS}")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${REL_LINK_FLAGS}")
        set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /LTCG")
    endif()

    # Ignore annoying DELAYLOAD warnings
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /ignore:4199")

    if(SLANG_MSVC_W4)
        if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
            string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
        endif()
    endif()

    if(SLANG_MSVC_STATIC_RUNTIME)
        set(variables
            CMAKE_C_FLAGS_DEBUG
            CMAKE_C_FLAGS_MINSIZEREL
            CMAKE_C_FLAGS_RELEASE
            CMAKE_C_FLAGS_RELWITHDEBINFO
            CMAKE_CXX_FLAGS_DEBUG
            CMAKE_CXX_FLAGS_MINSIZEREL
            CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_RELWITHDEBINFO
        )
        foreach(variable ${variables})
            if(${variable} MATCHES "/MD")
                string(REGEX REPLACE "/MD" "/MT" ${variable} "${${variable}}")
            endif()
        endforeach()
    endif()

    set(SLANG_WARN_FLAGS "/WX")
else()
    # Always include debug info
    add_compile_options(-g)

    # Color in diagnostics please
    if(CMAKE_GENERATOR MATCHES "Ninja")
        if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            add_compile_options(-Xclang -fcolor-diagnostics)
        else()
            add_compile_options(-fdiagnostics-color)
        endif()
    endif()

    if(CMAKE_BUILD_TYPE MATCHES "Debug")
        add_compile_options(-DDEBUG)
    endif()

    if(SLANG_COVERAGE)
        if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
            add_compile_options(-fprofile-instr-generate -fcoverage-mapping)
            link_libraries(-fprofile-instr-generate -fcoverage-mapping)
        else()
            add_compile_options(
                --coverage
                -fno-omit-frame-pointer
                -fno-optimize-sibling-calls
                -fno-inline
                -fno-inline-small-functions
                -fno-default-inline
            )
            link_libraries(--coverage)
        endif()
    endif()

    if(SLANG_SANITIZERS)
        add_compile_options("-fsanitize=${SLANG_SANITIZERS}")
        link_libraries("-fsanitize=${SLANG_SANITIZERS}")
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(SLANG_WARN_FLAGS
        -Wall
        -Wextra
        -Werror
        -Warray-bounds-pointer-arithmetic
        -Wassign-enum
        -Wbad-function-cast
        -Wcast-qual
        -Wchar-subscripts
        -Wcomma
        -Wconditional-uninitialized
        -Wconversion
        -Wdelete-non-virtual-dtor
        -Wdeprecated
        -Wduplicate-enum
        -Wduplicate-method-arg
        -Wembedded-directive
        -Wfor-loop-analysis
        -Wformat-pedantic
        -Widiomatic-parentheses
        -Wimplicit-fallthrough
        -Wpedantic
        -Wrange-loop-analysis
        -Wredundant-parens
        -Wreserved-id-macro
        -Wshadow
        -Wundefined-reinterpret-cast
        -Wunreachable-code-aggressive
        -Wzero-as-null-pointer-constant
        -Wno-missing-braces
    )
    if (APPLE)
        set(CMAKE_SHARED_LIBRARY_CREATE_CXX_FLAGS "${CMAKE_SHARED_LIBRARY_CREATE_CXX_FLAGS} -undefined dynamic_lookup")
        if(STATIC_BUILD)
          message(SEND_ERROR "STATIC_BUILD=ON not supported on macOS." )
        endif()
    endif()
elseif (CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(SLANG_WARN_FLAGS
        -Wall
        -Wextra
        -Werror
        -Wunused-value
        -Wformat-security
        -Wimplicit-fallthrough=5
        -Walloc-zero
        -Wlogical-op
        -Wlogical-not-parentheses
        -Wvla
        -Wduplicated-cond
        -Wtype-limits
        -Wno-maybe-uninitialized
    )
endif()

if(CMAKE_GENERATOR MATCHES "Visual Studio")
    # Only MSBuild needs this, other generators will compile one file at a time
    add_compile_options("/MP")
endif()

add_subdirectory(source)
add_subdirectory(tools)

if(SLANG_INCLUDE_TESTS)
    include(CTest)

    add_subdirectory(tests/regression)
    add_subdirectory(tests/unittests)

    if(SLANG_INCLUDE_LLVM)
        add_subdirectory(tests/simtests)
    endif()
endif()

if(SLANG_INCLUDE_DOCS)
    add_subdirectory(docs)
endif()
