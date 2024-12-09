cmake_minimum_required(VERSION 3.24)

if(${CMAKE_VERSION} VERSION_GREATER_EQUAL 3.30)
    cmake_policy(SET CMP0167 NEW)
endif()

project(workspace
    VERSION 1.5.0
    DESCRIPTION "workspace mechanism"
    HOMEPAGE_URL "https://github.com/holgerBerger/hpc-workspace/"
    LANGUAGES C CXX
)
add_definitions("-DWS_VERSION=\"${workspace_VERSION}\"")

message(STATUS "build type: " ${CMAKE_BUILD_TYPE})
message(STATUS "use cmake -DCMAKE_BUILD_TYPE=debug for debug build")

add_compile_options(-Wall -std=c++11)
add_link_options($<$<CONFIG:Release>:-s>)
add_link_options($<$<CONFIG:MinSizeRel>:-s>)

OPTION(STATIC "static linking" FALSE)
IF (STATIC)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static")
    set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++ -static")
    SET(EXTRA_STATIC_LIBS "pthread")
#    set(Boost_USE_STATIC_LIBS ON)
#    set(Boost_USE_STATIC_RUNTIME ON) 
ELSE (STATIC)
    set(Boost_USE_STATIC_LIBS OFF)
    set(Boost_USE_STATIC_RUNTIME OFF) 
ENDIF (STATIC)

OPTION(CHECK_ALL_GROUPS "search all groups for ACLs, not just primary" FALSE)
IF (CHECK_ALL_GROUPS)
    ADD_DEFINITIONS("-DCHECK_ALL_GROUPS")
ENDIF (CHECK_ALL_GROUPS)

OPTION(SETUID "use setuid instead of capabilites" TRUE)
IF (SETUID)
	ADD_DEFINITIONS(-DSETUID)
	SET(CAP "")
ELSE (SETUID)
    SET(CAP "cap")
ENDIF (SETUID)

OPTION(LUACALLOUTS "Enable LUA Callouts" FALSE)
IF (LUACALLOUTS)
	ADD_DEFINITIONS(-DLUACALLOUTS)
	find_package(Lua51 REQUIRED)
	INCLUDE_DIRECTORIES(${LUA_INCLUDE_DIR})
	LINK_DIRECTORIES(${LUA_LIBRARY_DIRS})
ENDIF (LUACALLOUTS)

set(Boost_USE_MULTITHREADED OFF)
option(USE_BOOST_REGEXP "use boost regexp instead of std::regexp" OFF)
if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
	message(STATUS "using boost_regexp for broken std::regex in gcc < gcc-4.9")
	set(USE_BOOST_REGEXP ON)
endif()

list(APPEND BOOST_COMPONENTS system filesystem program_options)
if(USE_BOOST_REGEXP)
    list(APPEND BOOST_COMPONENTS regex)
    add_definitions("-DUSE_BOOST_REGEXP")
endif()
find_package(Boost COMPONENTS ${BOOST_COMPONENTS} REQUIRED)


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

FIND_LIBRARY(YAML libyaml-cpp.so)
IF (YAML)
    MESSAGE("-- Found system YAML")
ELSE (YAML)
    MESSAGE("-- No system YAML, building own YAML")
	set(YAML_CPP_BUILD_TESTS OFF CACHE INTERNAL "" FORCE)
	#ADD_SUBDIRECTORY(${workspace_SOURCE_DIR}/yaml-cpp-release-0.5.3)
	#INCLUDE_DIRECTORIES(${workspace_SOURCE_DIR}/yaml-cpp-release-0.5.3/include)
	ADD_SUBDIRECTORY(${workspace_SOURCE_DIR}/yaml-cpp-yaml-cpp-0.6.3/)
    INCLUDE_DIRECTORIES(${workspace_SOURCE_DIR}/yaml-cpp-yaml-cpp-0.6.3//include)

ENDIF (YAML)

get_directory_property(LINKER_VAR LINK_DIRECTORIES)
message(STATUS "LINKER_VAR: ${LINKER_VAR}")

IF (IS_DIRECTORY "${PROJECT_SOURCE_DIR}/.git")
    add_definitions(-DIS_GIT_REPOSITORY)
    # git commit hash macro
    execute_process(
       COMMAND git log -1 --format=%h
       WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
       OUTPUT_VARIABLE GIT_COMMIT_HASH
       OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    add_definitions("-DGIT_COMMIT_HASH=\"${GIT_COMMIT_HASH}\"")
ENDIF ()

SET (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${workspace_BINARY_DIR}/bin)


add_executable(ws_allocate)
target_sources(ws_allocate PRIVATE
    ${workspace_SOURCE_DIR}/src/ws_allocate.cpp
    ${workspace_SOURCE_DIR}/src/ws.cpp
    ${workspace_SOURCE_DIR}/src/wsdb.cpp
)

add_executable(ws_release)
target_sources(ws_release PRIVATE
    ${workspace_SOURCE_DIR}/src/ws_release.cpp
    ${workspace_SOURCE_DIR}/src/ws.cpp
    ${workspace_SOURCE_DIR}/src/wsdb.cpp
)

add_library(ruh STATIC)
target_sources(ruh PRIVATE
    ${workspace_SOURCE_DIR}/src/ruh.cpp
)
target_link_libraries(ruh PUBLIC ${TLIB})

add_executable(ws_restore)
target_sources(ws_restore PRIVATE
    ${workspace_SOURCE_DIR}/src/ws_restore.cpp
    ${workspace_SOURCE_DIR}/src/ws.cpp
    ${workspace_SOURCE_DIR}/src/wsdb.cpp
)

TARGET_LINK_LIBRARIES( ws_allocate "-L${LINKER_VAR}" ${Boost_LIBRARIES} ${LUA_LIBRARIES} ${CAP} yaml-cpp ${EXTRA_STATIC_LIBS})
TARGET_LINK_LIBRARIES( ws_release "-L${LINKER_VAR}" ${Boost_LIBRARIES} ${LUA_LIBRARIES} ${CAP} yaml-cpp ${EXTRA_STATIC_LIBS})
TARGET_LINK_LIBRARIES( ws_restore "-L${LINKER_VAR}" ${Boost_LIBRARIES} ${LUA_LIBRARIES} ${CAP} yaml-cpp ruh ${EXTRA_STATIC_LIBS})


# Get install target
set(PROGRAM_PERMISSIONS_DEFAULT
    OWNER_WRITE OWNER_READ OWNER_EXECUTE
    GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE)
install(
    FILES
        bin/ws_extend
        bin/ws_find
        bin/ws_list
        bin/ws_register
        bin/ws_send_ical
        bin/ws_share
    DESTINATION bin
    PERMISSIONS ${PROGRAM_PERMISSIONS_DEFAULT}
)
install(
    TARGETS
        ws_allocate
        ws_release
        ws_restore
    DESTINATION bin
    PERMISSIONS ${PROGRAM_PERMISSIONS_DEFAULT}
    SETUID
)
install(
    FILES
        sbin/ws_expirer
        sbin/ws_restore
        sbin/ws_validate_config
    DESTINATION sbin
    PERMISSIONS ${PROGRAM_PERMISSIONS_DEFAULT}
)
install(
    FILES
        man/man1/ws_allocate.1
        man/man1/ws_find.1
        man/man1/ws_register.1
        man/man1/ws_restore.1
        man/man1/ws_extend.1
        man/man1/ws_list.1
        man/man1/ws_release.1
        man/man1/ws_send_ical.1
    DESTINATION share/man/man1
)

include(CPack)
