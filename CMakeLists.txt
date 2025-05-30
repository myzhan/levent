cmake_minimum_required (VERSION 2.8.12...3.15.0)
project(Levent)

set(levent_cmake "config.cmake")
if (EXISTS $ENV{LEVENT_CMAKE})
    set(levent_cmake $ENV{LEVENT_CMAKE})
endif()
include(${levent_cmake})

#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -ggdb")
#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -save-temps")

#message("name:${CMAKE_SYSTEM_NAME}")
#message("linker flags:${CMAKE_SHARED_LINKER_FLAGS}")
#message("compiler: ${CMAKE_C_COMPILER}")
include_directories(${LUA_INCLUDE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/deps/http-parser)
include_directories(${LUA_INCLUDE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/deps/libev)
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/deps/libev/.libs/)
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(CMAKE_SHARED_LINKER_FLAGS "-undefined dynamic_lookup")
endif()

add_definitions(-DHTTP_PARSER_STRICT=0)
if(WIN32)
    set(suffix ".dll")
    add_definitions(-DLUA_BUILD_AS_DLL -DLUA_LIB -DHTTP_PARSER_STRICT=0)
else()
    set(suffix ".so")
endif()

if(APPLE)
    set(OPENSSL_INCLUDE "/usr/local/opt/openssl/include")
    set(OPENSSL_LIB "/usr/local/opt/openssl/lib")
    execute_process(
            COMMAND which brew
            OUTPUT_VARIABLE BREW_PATH
            ERROR_QUIET
            OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    if(BREW_PATH)
        # 如果 brew 存在，尝试用 brew 查找 openssl，如果存在则修正默认路径
        execute_process(
                COMMAND brew --prefix openssl
                OUTPUT_VARIABLE OPENSSL_PREFIX
                ERROR_QUIET
                OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        if(OPENSSL_PREFIX)
            set(OPENSSL_INCLUDE ${OPENSSL_PREFIX}/include)
            set(OPENSSL_LIB ${OPENSSL_PREFIX}/lib)
        endif()
    endif()

    include_directories(${LUA_INCLUDE_DIR} ${OPENSSL_INCLUDE})
    set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH};${OPENSSL_LIB})
endif()

macro (add_lua_library lname)
    add_library(${lname} SHARED ${ARGN})
    if(WIN32)
        target_link_libraries(${lname} ${LUA_LIBRARIES})
    endif()
    set_target_properties(${lname} PROPERTIES PREFIX "")
    set_target_properties(${lname} PROPERTIES SUFFIX ${suffix})
endmacro()

# levent.module
set(CSOURCE src/lua-socket.c src/lua-errno.c src/lua-levent.c src/lua-ev.c src/lua-http-parser.c deps/http-parser/http_parser.c)
set(CMONGO cext/lua-mongo.c)
set(CBSON  cext/lua-bson.c)
set(CRYPTO cext/luacrypto/lcrypto.c)
find_library(CRYPTOLIB NAMES crypto)

IF(NOT WIN32)
    add_lua_library(levent ${CSOURCE})
    # ignore compile warning: incompatible-pointer-types
    target_compile_options(levent PRIVATE -Wno-incompatible-pointer-types)

    target_link_libraries(levent ev)
    add_custom_command(TARGET levent
        PRE_BUILD
        COMMAND CFLAGS=-fPIC ./configure --enable-shared=no
        COMMAND make
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/deps/libev)
    add_lua_library(mongo   ${CMONGO})
    add_lua_library(bson    ${CBSON})
    add_lua_library(crypto  ${CRYPTO})
    target_link_libraries(crypto ${CRYPTOLIB})
ELSE()
    set(CSOURCE ${CSOURCE} src/evwrap.c)
    add_lua_library(levent  ${CSOURCE})
    add_lua_library(mongo   ${CMONGO})
    add_lua_library(bson    ${CBSON})
    add_lua_library(crypto  ${CRYPTO})
    target_link_libraries(crypto ${CRYPTOLIB})
ENDIF()
