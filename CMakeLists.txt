#----------------------------------------------------------------------
#
#  Osmium Tool CMakeLists.txt
#
#----------------------------------------------------------------------

cmake_minimum_required(VERSION 2.8.5)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")


#----------------------------------------------------------------------
#
#  Projection version
#
#----------------------------------------------------------------------

project(osmium)

set(OSMIUM_VERSION_MAJOR 0)
set(OSMIUM_VERSION_MINOR 0)
set(OSMIUM_VERSION_PATCH 1)

set(OSMIUM_VERSION ${OSMIUM_VERSION_MAJOR}.${OSMIUM_VERSION_MINOR}.${OSMIUM_VERSION_PATCH})


#----------------------------------------------------------------------
#
#  Find external dependencies
#
#----------------------------------------------------------------------
find_package(Threads REQUIRED)
list(APPEND LIBRARIES ${CMAKE_THREAD_LIBS_INIT})

find_package(ZLIB REQUIRED)
include_directories(${ZLIB_INCLUDE_DIRS})
list(APPEND LIBRARIES ${ZLIB_LIBRARIES})

find_package(BZip2 REQUIRED)
include_directories(${BZIP2_INCLUDE_DIRS})
list(APPEND LIBRARIES ${BZIP2_LIBRARIES})

find_package(EXPAT REQUIRED)
include_directories(${EXPAT_INCLUDE_DIRS})
list(APPEND LIBRARIES ${EXPAT_LIBRARIES})

find_package(Protobuf REQUIRED)
include_directories(${PROTOBUF_INCLUDE_DIRS})
list(APPEND LIBRARIES ${PROTOBUF_LITE_LIBRARIES})

find_package(Boost COMPONENTS program_options REQUIRED)
include_directories(${BOOST_INCLUDE_DIRS})
list(APPEND LIBRARIES ${Boost_LIBRARIES})

find_package(Cryptopp REQUIRED)
include_directories(${CRYPTOPP_INCLUDE_DIRS})
list(APPEND LIBRARIES ${CRYPTOPP_LIBRARIES})

find_package(OSMPBF REQUIRED)
include_directories(${OSMPBF_INCLUDE_DIRS})
list(APPEND LIBRARIES ${OSMPBF_LIBRARIES})

find_package(Osmium REQUIRED)
include_directories(${OSMIUM_INCLUDE_DIRS})


#----------------------------------------------------------------------
#
#  Optional "cppcheck" target that checks C++ code
#
#----------------------------------------------------------------------
message(STATUS "Looking for cppcheck")
find_program(CPPCHECK cppcheck)

if(CPPCHECK)
    message(STATUS "Looking for cppcheck - found")
    set(CPPCHECK_OPTIONS --enable=warning,style,performance,portability,information,missingInclude)

    # cpp doesn't find system includes for some reason, suppress that report
    set(CPPCHECK_OPTIONS ${CPPCHECK_OPTIONS} --suppress=missingIncludeSystem)

    add_custom_target(cppcheck ${CPPCHECK} --std=c++11 ${CPPCHECK_OPTIONS} ${CMAKE_SOURCE_DIR}/src/*pp)
else()
    message(STATUS "Looking for cppcheck - not found")
    message(STATUS "  Make target cppcheck not available")
endif(CPPCHECK)


#----------------------------------------------------------------------
#
#  Optional "man" target to generate man pages
#
#----------------------------------------------------------------------
message(STATUS "Looking for pandoc")
find_program(PANDOC pandoc)

if(PANDOC)
    message(STATUS "Looking for pandoc - found")
    message(STATUS "  Manual pages will be built")
    execute_process(COMMAND date "+%Y-%m-%d" OUTPUT_VARIABLE PUBDATE OUTPUT_STRIP_TRAILING_WHITESPACE)
    set(PANDOC_MAN_OPTIONS -s -t man --template ${CMAKE_CURRENT_SOURCE_DIR}/doc/manpage.template --variable "description=osmium/${OSMIUM_VERSION}" --variable "date=${PUBDATE}")
    set(PANDOC_HTML_OPTIONS -s -t html)

    set(MAN_PAGES_1 osmium.1 osmium-apply-changes.1 osmium-cat.1 osmium-fileinfo.1 osmium-merge-changes.1 osmium-time-filter.1)
    set(MAN_PAGES_5 osmium-file-formats.5)
    set(MAN_NAMES ${MAN_PAGES_1} ${MAN_PAGES_5})
    set(MAN_FILES)
    foreach(m IN LISTS MAN_NAMES)
        set(mf ${CMAKE_BINARY_DIR}/${m})
        string(REGEX REPLACE ".[0-9]\$" "" mws "${m}")
        set(ms ${CMAKE_SOURCE_DIR}/doc/${mws}.md)
        add_custom_command(OUTPUT ${mf}
            COMMAND ${PANDOC} ${PANDOC_MAN_OPTIONS} -o ${mf} ${ms}
            DEPENDS ${ms}
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
            COMMENT "Building manpage ${mf}"
            VERBATIM)
        list(APPEND MAN_FILES ${mf})
    endforeach()

    add_custom_target(man ALL DEPENDS ${MAN_FILES})
else()
    message(STATUS "Looking for pandoc - not found")
    message(STATUS "  Manual pages will not be built")
endif(PANDOC)


#----------------------------------------------------------------------
#
#  Create src/config.hpp
#
#----------------------------------------------------------------------
configure_file(
    ${PROJECT_SOURCE_DIR}/src/config.hpp.in
    ${PROJECT_BINARY_DIR}/src/config.hpp
)

include_directories(${PROJECT_BINARY_DIR}/src)


#----------------------------------------------------------------------
#
#  C++ compilation options
#
#----------------------------------------------------------------------
add_definitions(-D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64)

add_compile_options(-std=c++11)

add_compile_options(
    -Wall
    -Wextra
    -pedantic
    -Wredundant-decls
    -Wdisabled-optimization
    -Wctor-dtor-privacy
    -Wnon-virtual-dtor
    -Woverloaded-virtual
    -Wsign-promo
    -Wold-style-cast
)

# always compile with optimization, even in debug mode
set(CMAKE_CXX_FLAGS_DEBUG -O3)
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O3 -g")

if(APPLE)
    add_compile_options(-stdlib=libc++)
    set(LDFLAGS ${LDFLAGS} -stdlib=libc++)
endif(APPLE)


#----------------------------------------------------------------------

add_subdirectory(src)

install(FILES README.md DESTINATION share/doc/osmium)


#----------------------------------------------------------------------