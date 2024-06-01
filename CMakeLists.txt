cmake_minimum_required(VERSION 3.21)

project(CppInterOp LANGUAGES CXX)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -fno-rtti")

# Turn on using VS solution folders
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

find_package(LLVM REQUIRED CONFIG)
message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

find_package(Clang REQUIRED CONFIG)
message(STATUS "Found Clang ${Clang_PACKAGE_VERSION}")
message(STATUS "Using ClangConfig.cmake in: ${Clang_DIR}")

add_library(CppInterOp SHARED)
add_subdirectory(lib)
add_subdirectory(include)

target_include_directories(CppInterOp PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${LLVM_INCLUDE_DIRS}>
    $<BUILD_INTERFACE:${Clang_INCLUDE_DIRS}>
    $<INSTALL_INTERFACE:include>)
    
target_compile_features(CppInterOp PRIVATE cxx_std_17)

target_compile_definitions(CppInterOp PUBLIC "USE_REPL")
target_compile_definitions(CppInterOp PUBLIC "__STDC_FORMAT_MACROS") # see https://github.com/tensorflow/tensorflow/issues/12998

file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" CPPINTEROP_VERSION)
string(REGEX MATCH "([0-9]*)\.([0-9]*)\.([0-9]*)" CPPINTEROP_VERSION_ONLY "${CPPINTEROP_VERSION}")
set(CPPINTEROP_VERSION_MAJOR "${CMAKE_MATCH_1}")
set(CPPINTEROP_VERSION_MINOR "${CMAKE_MATCH_2}")
set(CPPINTEROP_VERSION_PATCH "${CMAKE_MATCH_3}")
message(STATUS "CppInterOp Version: ${CPPINTEROP_VERSION}")
message(STATUS "LLVM Binary Directory: ${LLVM_BINARY_DIR}")

target_compile_definitions(CppInterOp PRIVATE "CPPINTEROP_VERSION=\"${CPPINTEROP_VERSION}\"")
target_compile_definitions(CppInterOp PRIVATE "LLVM_BINARY_DIR=\"${LLVM_BINARY_DIR}\";")

target_link_libraries(CppInterOp PUBLIC LLVM clang-cpp)

# Install CompilationDatabase
set_target_properties(CppInterOp PROPERTIES EXPORT_COMPILE_COMMANDS true)
set(ccmds_json ${CMAKE_CURRENT_BINARY_DIR}/compile_commands.json)
if(EXISTS ${ccmds_json})
    message(STATUS "Found CompilationDatabase File: " ${ccmds_json})
    install(FILES ${ccmds_json} DESTINATION share)
endif()

# Install Binaries
install(TARGETS CppInterOp
        EXPORT CppInterOpTargets
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib/static
        INCLUDES DESTINATION include/clang)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/clang/ DESTINATION include/clang
        FILES_MATCHING PATTERN "*.h")

# Install CMake targets
install(EXPORT CppInterOpTargets
        NAMESPACE CppInterOp::
        FILE CppInterOpConfig.cmake
        DESTINATION lib/cmake/CppInterOp)
