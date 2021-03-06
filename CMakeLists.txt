project(appster)
cmake_minimum_required(VERSION 3.0.2)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")

##
# Options
##

option(BUILD_REDIS "enables or disables redis module" ON)
option(BUILD_SQL "enables or disables sql module" ON)
option(BUILD_SSL "enables or disables tls/ssl extenstion" ON)

##
# Dependencies
##

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

add_subdirectory(deps)
include_directories(SYSTEM deps)

find_package(Hiredis)
find_package(PostgreSQL)
find_package(OpenSSL)
find_package(libdill REQUIRED)
find_package(LibUV REQUIRED)

include_directories(
    ${HIREDIS_INCLUDE_DIR}
    ${PostgreSQL_INCLUDE_DIRS}
    ${LIBDILL_INCLUDE_DIRS}
    ${LIBUV_INCLUDE_DIRS}
    ${OPENSSL_INCLUDE_DIR}
)

if(HIREDIS_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${HIREDIS_DEFINITIONS}")
endif()

if (OPENSSL_FOUND)
    add_definitions(-DHAS_CRYPTO)
endif()

##
# Files
##

set(SRC_LIST
    src/appster.c
    src/format.c
    src/log.c
    src/schema.c
)

if (OPENSSL_FOUND)
    set(SRC_LIST ${SRC_LIST} src/crypto.c)
endif()

if(HIREDIS_FOUND)
    set(SRC_LIST ${SRC_LIST} src/module/redis.c)
endif()

if(PostgreSQL_FOUND)
    set(SRC_LIST ${SRC_LIST} src/module/sql.c)
endif()

##
# Link
##

add_library(object_${PROJECT_NAME} OBJECT ${SRC_LIST})

set_property(TARGET object_${PROJECT_NAME} PROPERTY POSITION_INDEPENDENT_CODE 1)

add_library(shared_${PROJECT_NAME} SHARED $<TARGET_OBJECTS:object_${PROJECT_NAME}>)
add_library(static_${PROJECT_NAME} STATIC $<TARGET_OBJECTS:object_${PROJECT_NAME}>)

target_link_libraries(static_${PROJECT_NAME}
    deps
)
target_link_libraries(shared_${PROJECT_NAME}
    deps
)

target_link_libraries(static_${PROJECT_NAME}
    ${HIREDIS_LIBRARIES}
    ${PostgreSQL_LIBRARIES}
    ${LIBDILL_LIBRARIES}
    ${LIBUV_LIBRARIES}
    ${OPENSSL_LIBRARIES}
)

target_link_libraries(shared_${PROJECT_NAME}
    ${HIREDIS_LIBRARIES}
    ${PostgreSQL_LIBRARIES}
    ${LIBDILL_LIBRARIES}
    ${LIBUV_LIBRARIES}
    ${OPENSSL_LIBRARIES}
)

##
# Compiler & linker flags
##

set(CMAKE_C_FLAGS "-Wall -Wpedantic -Wno-sign-compare -Wno-variadic-macros")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0 -fsanitize=address -fno-omit-frame-pointer -Wno-unused-function")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address -static-libasan")
#    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0")
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
endif()

##
# Examples
##

add_executable(example_read src/examples/read.c)
add_executable(example_write src/examples/write.c)

target_link_libraries(example_read static_${PROJECT_NAME} deps)
target_link_libraries(example_write static_${PROJECT_NAME} deps)

if(HIREDIS_FOUND)
    add_executable(example_redis
        src/examples/redis.c
    )
    target_link_libraries(example_redis
        static_${PROJECT_NAME}
        deps
    )
endif()

if(PostgreSQL_FOUND)
    add_executable(example_sql
        src/examples/sql.c
    )
    target_link_libraries(example_sql
        static_${PROJECT_NAME}
        deps
    )
endif()

##
# Installation
##
set (LIB_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/lib")
set (INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include/appster")

install(TARGETS shared_${PROJECT_NAME} DESTINATION "${LIB_INSTALL_DIR}")
install(TARGETS static_${PROJECT_NAME} DESTINATION "${LIB_INSTALL_DIR}")
install(FILES src/appster.h DESTINATION ${INCLUDE_INSTALL_DIR})

if(HIREDIS_FOUND)
    install(FILES src/module/redis.h DESTINATION "${INCLUDE_INSTALL_DIR}/module")
endif()

if(PostgreSQL_FOUND)
    install(FILES src/module/sql.h DESTINATION "${INCLUDE_INSTALL_DIR}/module")
endif()
