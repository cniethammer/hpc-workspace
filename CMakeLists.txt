cmake_minimum_required(VERSION 3.18)

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL 3.20)
    # enforce explicit source file extensions
    cmake_policy(SET CMP0115 NEW)
endif()

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL 3.30)
    # prefer to use the upstream BoostConfig.cmake in favour of FindBoost
    cmake_policy(SET CMP0167 NEW)
endif()

project(workspace
    VERSION 1.5.0
    DESCRIPTION "workspace mechanism"
    HOMEPAGE_URL "https://github.com/holgerBerger/hpc-workspace"
    LANGUAGES C CXX
)

message(STATUS "build type: " ${CMAKE_BUILD_TYPE})
message(STATUS "use cmake -DCMAKE_BUILD_TYPE=debug for debug build")


set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${workspace_BINARY_DIR}/bin)

add_subdirectory(src/)


# Installion
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
    PERMISSIONS ${PROGRAM_PERMISSIONS_DEFAULT} SETUID
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
        man/man1/ws_extend.1
        man/man1/ws_find.1
        man/man1/ws_list.1
        man/man1/ws_register.1
        man/man1/ws_restore.1
        man/man1/ws_release.1
        man/man1/ws_send_ical.1
    DESTINATION share/man/man1
)
