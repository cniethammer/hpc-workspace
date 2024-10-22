add_compile_definitions("-DWS_VERSION=\"${PROJECT_VERSION}\"")
add_link_options($<$<CONFIG:Release>:-s>)
add_link_options($<$<CONFIG:MinSizeRel>:-s>)

option(STATIC "static linking" FALSE)
if(STATIC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static")
    set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static")
endif()

option(CHECK_ALL_GROUPS "search all groups for ACLs, not just primary" FALSE)
if(CHECK_ALL_GROUPS)
    add_definitions("-DCHECK_ALL_GROUPS")
endif()

if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
	message(STATUS "using boost_regexp for broken gcc")
	set(USE_BOOST_REGEXP on)
ELSE ()
	message(STATUS "using C++11 regexp")
endif()

option(SETUID "use setuid instead of capabilites" TRUE)
if(SETUID)
    message(STATUS "Using setuid instead of capabilities")
	add_definitions(-DSETUID)
endif()


# find terminfo for "are you human" checker
FIND_LIBRARY(TERMINFO NAMES libtinfo.so libtinfo.a)
IF (TERMINFO)
    MESSAGE("-- Found terminfo")
    INCLUDE_DIRECTORIES(${TERMINFO_INCLUDE_DIRS})
    SET(LIBS ${LIBS} ${TERMINFO_LIBRARIES})
    SET(TLIB "tinfo")
ELSE (TERMINFO)
    MESSAGE("-- No terminfo, trying curses")
    #find curses for terminfo for "are you human" checker
    SET(CURSES_NEED_NCURSES TRUE)
    find_package(Curses REQUIRED)
    include_directories(${CURSES_INCLUDE_DIR})
    link_directories(${CURSES_LIBRARY_DIR})
    SET(TLIB ${CURSES_LIBRARIES})
ENDIF (TERMINFO)


if(IS_DIRECTORY "${PROJECT_SOURCE_DIR}/.git")
    add_definitions(-DIS_GIT_REPOSITORY)
    # git commit hash macro
    execute_process(
       COMMAND git log -1 --format=%h
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
       OUTPUT_VARIABLE GIT_COMMIT_HASH
       OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    add_definitions("-DGIT_COMMIT_HASH=\"${GIT_COMMIT_HASH}\"")
    message(STATUS "Building from git repositroy (hash: $GIT_COMMIT_HASH)")
endif()


# Boost setup
set(Boost_USE_MULTITHREADED OFF)
if(STATIC)
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME ON)
else()
    set(Boost_USE_STATIC_LIBS OFF)
    set(Boost_USE_STATIC_RUNTIME OFF)
endif()

option(USE_BOOST_REGEXP "use boost regexp in case std::regexp is broken (rh7 gcc)" FALSE)
if(CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
	message(STATUS "using boost_regexp instead of broken std::regex in gcc < gcc-4.9")
	set(USE_BOOST_REGEXP ON)
endif()
if(USE_BOOST_REGEXP)
    add_definitions("-DUSE_BOOST_REGEXP")
endif()

find_package(Boost COMPONENTS filesystem program_options system REQUIRED)
if(USE_BOOST_REGEXP)
	find_package(Boost COMPONENTS regex REQUIRED)
endif()


# yaml-cpp setup
find_package(yaml-cpp QUIET)
if(NOT ${yaml-cpp_FOUND})
    message(STATUS "yaml-cpp not found, building from source ...")
    include(FetchContent)
    FetchContent_Declare(
        yaml-cpp
        GIT_REPOSITORY https://github.com/jbeder/yaml-cpp.git
        GIT_TAG 0.8.0 # Can be a tag (yaml-cpp-x.x.x), a commit hash, or a branch name (master)
    )
    FetchContent_MakeAvailable(yaml-cpp)
endif()


# Lua setup
option(LUACALLOUTS "Enable LUA Callouts" FALSE)
if(LUACALLOUTS)
	add_definitions("-DLUACALLOUTS")
	find_package(Lua REQUIRED)
endif()

add_library(ws)
target_sources(ws PRIVATE
    ws.cpp
)
target_link_libraries(ws
    PUBLIC
        Boost::program_options
        Boost::filesystem
        yaml-cpp::yaml-cpp
)
add_library(wsdb)
target_sources(wsdb PRIVATE
    wsdb.cpp
)
target_link_libraries(wsdb
    PUBLIC
        Boost::program_options
        yaml-cpp::yaml-cpp
)

add_executable(ws_allocate)
target_compile_features(ws_allocate PUBLIC cxx_std_11)
target_sources(ws_allocate PRIVATE
	ws_allocate.cpp
)
target_link_libraries(ws_allocate
    PRIVATE
        ws
        wsdb
)

add_executable(ws_release)
target_sources(ws_release PRIVATE
    ws_release.cpp
)
target_link_libraries(ws_release
    PRIVATE
        ws
        wsdb
)

add_executable(ws_restore)
target_sources(ws_restore PRIVATE
    ws_release.cpp
)

target_link_libraries(ws_restore
    PRIVATE
        ws
        wsdb
)
