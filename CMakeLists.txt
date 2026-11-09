cmake_minimum_required(VERSION 3.16)
project(Slide LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find Java (needed to run ANTLR tool if antlr4 binary is not directly available)
find_package(Java COMPONENTS Runtime)

# Locate antlr4 executable or jar
find_program(ANTLR_EXECUTABLE NAMES antlr4 antlr)

# Define ANTLR generation command
set(ANTLR_INPUT ${CMAKE_CURRENT_SOURCE_DIR}/Slide.g4)
set(ANTLR_GEN_DIR ${CMAKE_CURRENT_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${ANTLR_GEN_DIR})

set(ANTLR_OUTPUTS
    ${ANTLR_GEN_DIR}/SlideLexer.cpp
    ${ANTLR_GEN_DIR}/SlideLexer.h
    ${ANTLR_GEN_DIR}/SlideParser.cpp
    ${ANTLR_GEN_DIR}/SlideParser.h
    ${ANTLR_GEN_DIR}/SlideBaseVisitor.cpp
    ${ANTLR_GEN_DIR}/SlideBaseVisitor.h
    ${ANTLR_GEN_DIR}/SlideVisitor.cpp
    ${ANTLR_GEN_DIR}/SlideVisitor.h
)

if(ANTLR_EXECUTABLE)
    set(ANTLR_CMD ${ANTLR_EXECUTABLE})
elseif(Java_JAVA_EXECUTABLE AND EXISTS "/usr/share/java/antlr-complete.jar")
    set(ANTLR_CMD ${Java_JAVA_EXECUTABLE} -jar /usr/share/java/antlr-complete.jar)
else()
    set(ANTLR_CMD antlr4)
endif()

add_custom_command(
    OUTPUT ${ANTLR_OUTPUTS}
    COMMAND ${ANTLR_CMD} -Dlanguage=Cpp -visitor -no-listener -o ${ANTLR_GEN_DIR} ${ANTLR_INPUT}
    DEPENDS ${ANTLR_INPUT}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating ANTLR4 C++ parser files from Slide.g4"
)

# Find ANTLR4 C++ runtime
find_path(ANTLR4_INCLUDE_DIR antlr4-runtime.h
    PATH_SUFFIXES antlr4-runtime
    PATHS /usr/include/antlr4-runtime /usr/local/include/antlr4-runtime
)

find_library(ANTLR4_LIBRARY NAMES antlr4-runtime)

if(NOT ANTLR4_LIBRARY OR NOT ANTLR4_INCLUDE_DIR)
    message(STATUS "Looking for antlr4-runtime via pkg-config...")
    find_package(PkgConfig QUIET)
    if(PKG_CONFIG_FOUND)
        pkg_check_modules(ANTLR4 QUIET antlr4-runtime)
    endif()
endif()

add_executable(slide
    Main.cpp
    ${ANTLR_OUTPUTS}
)

target_include_directories(slide PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${ANTLR_GEN_DIR}
    ${ANTLR4_INCLUDE_DIR}
    ${ANTLR4_INCLUDE_DIRS}
)

if(ANTLR4_LIBRARY)
    target_link_libraries(slide PRIVATE ${ANTLR4_LIBRARY})
elseif(ANTLR4_LIBRARIES)
    target_link_libraries(slide PRIVATE ${ANTLR4_LIBRARIES})
else()
    target_link_libraries(slide PRIVATE antlr4-runtime)
endif()

