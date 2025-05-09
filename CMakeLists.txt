cmake_minimum_required(VERSION 3.18.0)

project(GLFM C)

option(GLFM_BUILD_EXAMPLES "Build the GLFM examples" OFF)
option(GLFM_USE_CLANG_TIDY "Use Clang Tidy when building (Android and Emscripten only)" OFF)

set(GLFM_HEADERS include/glfm.h)

if (CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
    set(GLFM_SRC src/glfm_internal.h src/glfm_emscripten.c)
    set(GLFM_COMPILE_OPTIONS -Wno-gnu-zero-variadic-macro-arguments -Wno-dollar-in-identifier-extension
        -Wno-c23-extensions -Wno-pre-c11-compat)
elseif (CMAKE_SYSTEM_NAME STREQUAL "Android")
    set(GLFM_SRC src/glfm_internal.h src/glfm_android.c)
elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    if (${CMAKE_OSX_SYSROOT} MATCHES "(MacOS)+")
        set(CMAKE_OSX_SYSROOT "iphoneos")
    endif()

    set(GLFM_SRC src/glfm_internal.h src/glfm_apple.m)
    set(GLFM_COMPILE_OPTIONS -Wno-auto-import -Wno-direct-ivar-access)
else()
    message(FATAL_ERROR "CMAKE_SYSTEM_NAME ('${CMAKE_SYSTEM_NAME}') expected to be Darwin, Emscripten, or Android")
endif()

if (GLFM_USE_CLANG_TIDY)
    # Android target: use clang-tidy from NDK.
    # Emscripten target: use clang-tidy on the host system.
    # On macOS host, try: $(brew --prefix llvm)/bin/clang-tidy
    find_program(CLANG_TIDY_EXECUTABLE clang-tidy)
    if ((NOT CLANG_TIDY_EXECUTABLE) AND (CMAKE_HOST_SYSTEM_NAME STREQUAL "Darwin"))
        execute_process(
            COMMAND brew --prefix llvm
            RESULT_VARIABLE BREW_LLVM
            OUTPUT_VARIABLE BREW_LLVM_PREFIX
            OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        if (BREW_LLVM EQUAL 0 AND EXISTS "${BREW_LLVM_PREFIX}")
            set(CLANG_TIDY_EXECUTABLE "${BREW_LLVM_PREFIX}/bin/clang-tidy")
        endif()
    endif()
    if (NOT CLANG_TIDY_EXECUTABLE)
        message(WARNING "GLFM_USE_CLANG_TIDY=ON but clang-tidy not found")
    else()
        if (CMAKE_SYSTEM_NAME STREQUAL "Android")
            set(CMAKE_C_CLANG_TIDY ${CLANG_TIDY_EXECUTABLE}
                --config-file=${PROJECT_SOURCE_DIR}/tests/clang-tidy-analyze.yml)
        elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
            message(WARNING "GLFM_USE_CLANG_TIDY=ON but clang-tidy not functional with Xcode generator")
        elseif (CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
            set(CMAKE_C_USE_RESPONSE_FILE_FOR_INCLUDES 0)
            set(CMAKE_C_CLANG_TIDY ${CLANG_TIDY_EXECUTABLE}
                --config-file=${PROJECT_SOURCE_DIR}/tests/clang-tidy-analyze.yml
                --extra-arg=--target=wasm32
                --extra-arg=-D__EMSCRIPTEN__
                --extra-arg=-isystem${EMSCRIPTEN_ROOT_PATH}/cache/sysroot/include)
        endif()
    endif()
endif()

add_library(glfm ${GLFM_SRC} ${GLFM_HEADERS})
target_include_directories(glfm PUBLIC include)
target_include_directories(glfm PRIVATE src)

source_group(include FILES ${GLFM_HEADERS})
source_group(src FILES ${GLFM_SRC})

set_target_properties(glfm PROPERTIES C_STANDARD 11)
if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    # Disable the -Wunsafe-buffer-usage warning because it reports false positives when bounds checking arrays, and
    # there appears to be no solution for C pointers.
    set_property(TARGET glfm PROPERTY COMPILE_OPTIONS -Weverything -Wwrite-strings -Wno-unknown-warning-option
        -Wno-padded -Wno-covered-switch-default -Wno-declaration-after-statement -Wno-unsafe-buffer-usage
        -Wno-missing-include-dirs
        ${GLFM_COMPILE_OPTIONS})
elseif (CMAKE_C_COMPILER_ID MATCHES "GNU")
    set_property(TARGET glfm PROPERTY COMPILE_OPTIONS -Wall -Wextra -Wpedantic -Wwrite-strings ${GLFM_COMPILE_OPTIONS})
elseif (CMAKE_C_COMPILER_ID MATCHES "MSVC")
    set_property(TARGET glfm PROPERTY COMPILE_OPTIONS /Wall ${GLFM_COMPILE_OPTIONS})
endif()

if (CMAKE_SYSTEM_NAME STREQUAL "Android")
    find_library(log-lib log)
    find_library(android-lib android)
    find_library(EGL-lib EGL)
    find_library(GLESv2-lib GLESv2)
    target_link_libraries(glfm ${log-lib} ${android-lib} ${EGL-lib} ${GLESv2-lib})
elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    target_compile_definitions(glfm PRIVATE GLES_SILENCE_DEPRECATION)
    set_target_properties(glfm PROPERTIES
        ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/GLFM.build/lib # For Archiving
        XCODE_ATTRIBUTE_SUPPORTED_PLATFORMS "iphoneos iphonesimulator appletvos appletvsimulator macosx"
        XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET  11.0  # Minimum supported by Xcode 14
        XCODE_ATTRIBUTE_TVOS_DEPLOYMENT_TARGET      11.0  # Minimum supported by Xcode 14
        XCODE_ATTRIBUTE_MACOSX_DEPLOYMENT_TARGET   10.13  # Minimum supported by Xcode 14
        XCODE_ATTRIBUTE_CLANG_ENABLE_MODULES         YES  # Automatically import frameworks

        # Override default warnings in Xcode because using "-Weverything" does not.
        # CMake 3.25.1, Xcode 14.2.
        XCODE_ATTRIBUTE_CLANG_WARN_ASSIGN_ENUM                            YES
        XCODE_ATTRIBUTE_CLANG_WARN_ATOMIC_IMPLICIT_SEQ_CST                YES
        XCODE_ATTRIBUTE_CLANG_WARN_BLOCK_CAPTURE_AUTORELEASING            YES
        XCODE_ATTRIBUTE_CLANG_WARN_BOOL_CONVERSION                        YES
        XCODE_ATTRIBUTE_CLANG_WARN_COMMA                                  YES
        XCODE_ATTRIBUTE_CLANG_WARN_COMPLETION_HANDLER_MISUSE              YES
        XCODE_ATTRIBUTE_CLANG_WARN_CONSTANT_CONVERSION                    YES
        XCODE_ATTRIBUTE_CLANG_WARN_CXX0X_EXTENSIONS                       YES
        XCODE_ATTRIBUTE_CLANG_WARN_DEPRECATED_OBJC_IMPLEMENTATIONS        YES
        XCODE_ATTRIBUTE_CLANG_WARN_DIRECT_OBJC_ISA_USAGE                  YES_ERROR
        XCODE_ATTRIBUTE_CLANG_WARN_DOCUMENTATION_COMMENTS                 YES
        XCODE_ATTRIBUTE_CLANG_WARN_EMPTY_BODY                             YES
        XCODE_ATTRIBUTE_CLANG_WARN_ENUM_CONVERSION                        YES
        XCODE_ATTRIBUTE_CLANG_WARN_FLOAT_CONVERSION                       YES
        XCODE_ATTRIBUTE_CLANG_WARN_FRAMEWORK_INCLUDE_PRIVATE_FROM_PUBLIC  YES
        XCODE_ATTRIBUTE_CLANG_WARN_IMPLICIT_FALLTHROUGH                   YES
        XCODE_ATTRIBUTE_CLANG_WARN_IMPLICIT_SIGN_CONVERSION               YES
        XCODE_ATTRIBUTE_CLANG_WARN_INFINITE_RECURSION                     YES
        XCODE_ATTRIBUTE_CLANG_WARN_INT_CONVERSION                         YES
        XCODE_ATTRIBUTE_CLANG_WARN_NON_LITERAL_NULL_CONVERSION            YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_EXPLICIT_OWNERSHIP_TYPE           YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_IMPLICIT_ATOMIC_PROPERTIES        YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_IMPLICIT_RETAIN_SELF              YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_INTERFACE_IVARS                   YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_LITERAL_CONVERSION                YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_MISSING_PROPERTY_SYNTHESIS        YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_REPEATED_USE_OF_WEAK              YES
        XCODE_ATTRIBUTE_CLANG_WARN_OBJC_ROOT_CLASS                        YES_ERROR
        XCODE_ATTRIBUTE_CLANG_WARN_QUOTED_INCLUDE_IN_FRAMEWORK_HEADER     YES
        XCODE_ATTRIBUTE_CLANG_WARN_RANGE_LOOP_ANALYSIS                    YES
        XCODE_ATTRIBUTE_CLANG_WARN_SEMICOLON_BEFORE_METHOD_BODY           YES
        XCODE_ATTRIBUTE_CLANG_WARN_STRICT_PROTOTYPES                      YES
        XCODE_ATTRIBUTE_CLANG_WARN_SUSPICIOUS_IMPLICIT_CONVERSION         YES
        XCODE_ATTRIBUTE_CLANG_WARN_SUSPICIOUS_MOVE                        YES
        XCODE_ATTRIBUTE_CLANG_WARN_UNGUARDED_AVAILABILITY                 YES_AGGRESSIVE
        XCODE_ATTRIBUTE_CLANG_WARN_UNREACHABLE_CODE                       YES
        XCODE_ATTRIBUTE_CLANG_WARN__ARC_BRIDGE_CAST_NONARC                YES
        XCODE_ATTRIBUTE_CLANG_WARN__DUPLICATE_METHOD_MATCH                YES
        XCODE_ATTRIBUTE_CLANG_WARN__EXIT_TIME_DESTRUCTORS                 YES
        XCODE_ATTRIBUTE_GCC_WARN_64_TO_32_BIT_CONVERSION                  YES_ERROR
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_DEPRECATED_FUNCTIONS               YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_INVALID_OFFSETOF_MACRO             YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_FIELD_INITIALIZERS         YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_NEWLINE                    YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_MISSING_PROTOTYPES                 YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_POINTER_SIGNEDNESS                 YES
        XCODE_ATTRIBUTE_GCC_WARN_ABOUT_RETURN_TYPE                        YES_ERROR
        XCODE_ATTRIBUTE_GCC_WARN_ALLOW_INCOMPLETE_PROTOCOL                YES
        XCODE_ATTRIBUTE_GCC_WARN_CHECK_SWITCH_STATEMENTS                  YES
        XCODE_ATTRIBUTE_GCC_WARN_FOUR_CHARACTER_CONSTANTS                 YES
        XCODE_ATTRIBUTE_GCC_WARN_HIDDEN_VIRTUAL_FUNCTIONS                 YES
        XCODE_ATTRIBUTE_GCC_WARN_INITIALIZER_NOT_FULLY_BRACKETED          YES
        XCODE_ATTRIBUTE_GCC_WARN_MISSING_PARENTHESES                      YES
        XCODE_ATTRIBUTE_GCC_WARN_NON_VIRTUAL_DESTRUCTOR                   YES
        XCODE_ATTRIBUTE_GCC_WARN_SHADOW                                   YES
        XCODE_ATTRIBUTE_GCC_WARN_SIGN_COMPARE                             YES
        XCODE_ATTRIBUTE_GCC_WARN_STRICT_SELECTOR_MATCH                    YES
        XCODE_ATTRIBUTE_GCC_WARN_TYPECHECK_CALLS_TO_PRINTF                YES
        XCODE_ATTRIBUTE_GCC_WARN_UNDECLARED_SELECTOR                      YES
        XCODE_ATTRIBUTE_GCC_WARN_UNINITIALIZED_AUTOS                      YES_AGGRESSIVE
        XCODE_ATTRIBUTE_GCC_WARN_UNKNOWN_PRAGMAS                          YES
        XCODE_ATTRIBUTE_GCC_WARN_UNUSED_FUNCTION                          YES
        XCODE_ATTRIBUTE_GCC_WARN_UNUSED_LABEL                             YES
        XCODE_ATTRIBUTE_GCC_WARN_UNUSED_PARAMETER                         YES
        XCODE_ATTRIBUTE_GCC_WARN_UNUSED_VALUE                             YES
        XCODE_ATTRIBUTE_GCC_WARN_UNUSED_VARIABLE                          YES
    )
endif()

if (GLFM_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
