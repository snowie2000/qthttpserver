# Standalone CMakeLists.txt for Qt5 QtHttpServer backport.
# Builds Qt5SslServer and Qt5HttpServer as independent libraries
# without requiring the full Qt source tree / qt_build_repo() infrastructure.
#
# Options:
#   BUILD_SHARED_LIBS=ON  (default) — build DLLs / .so
#   BUILD_SHARED_LIBS=OFF          — build static libs (.lib / .a)

cmake_minimum_required(VERSION 3.14)

project(QtHttpServer
    VERSION 5.12.0
    DESCRIPTION "Qt HTTP Server (Qt5 backport)"
    LANGUAGES CXX C
)

# ── Build settings ────────────────────────────────────────────────────────────
option(BUILD_SHARED_LIBS "Build shared libraries (DLL/so). Set OFF for static libs." ON)

set(CMAKE_AUTOMOC ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

# Put DLLs / import libs next to the source so tooling finds them easily
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")

# ── Find Qt5 ─────────────────────────────────────────────────────────────────
find_package(Qt5 5.12 REQUIRED COMPONENTS Core Network)
find_package(Qt5 OPTIONAL_COMPONENTS Concurrent WebSockets)

# Detect SSL support from the Qt5Network CMake package feature variables.
# Qt5.13+ sets Qt5Network_FEATURE_ssl; fall back to header detection for older.
if(DEFINED Qt5Network_FEATURE_ssl AND Qt5Network_FEATURE_ssl EQUAL 1)
    set(QT_HAVE_SSL TRUE)
elseif(NOT DEFINED Qt5Network_FEATURE_ssl)
    # Probe: look for qsslsocket.h inside the Qt includes
    get_target_property(_qt5_net_incs Qt5::Network INTERFACE_INCLUDE_DIRECTORIES)
    find_path(_qssl_inc "QtNetwork/qsslsocket.h" HINTS ${_qt5_net_incs} NO_DEFAULT_PATH)
    if(_qssl_inc)
        set(QT_HAVE_SSL TRUE)
    else()
        set(QT_HAVE_SSL FALSE)
    endif()
    unset(_qssl_inc CACHE)
else()
    set(QT_HAVE_SSL FALSE)
endif()

message(STATUS "Qt SSL support: ${QT_HAVE_SSL}")
message(STATUS "Qt Concurrent:  ${Qt5Concurrent_FOUND}")
message(STATUS "Qt WebSockets:  ${Qt5WebSockets_FOUND}")

# ── Private-header include bridge ─────────────────────────────────────────────
# The source files use the Qt convention  #include <private/xxx_p.h>.
# During a full Qt source build, syncqt creates the proper include tree.
# Here we replicate that by copying the _p.h files into a "private/"
# subdirectory inside the CMake binary dir and adding the parent to the path.

# --- SslServer ---
set(SSL_PRIV_ROOT "${CMAKE_BINARY_DIR}/qt_priv_inc/sslserver")
file(MAKE_DIRECTORY "${SSL_PRIV_ROOT}/private")
file(COPY
    "${CMAKE_SOURCE_DIR}/src/sslserver/qsslserver_p.h"
    DESTINATION "${SSL_PRIV_ROOT}/private")

# --- HttpServer ---
set(HTTP_PRIV_ROOT "${CMAKE_BINARY_DIR}/qt_priv_inc/httpserver")
file(MAKE_DIRECTORY "${HTTP_PRIV_ROOT}/private")
file(COPY
    "${CMAKE_SOURCE_DIR}/src/httpserver/qabstracthttpserver_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserver_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverliterals_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverrequest_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverresponder_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverresponse_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverrouter_p.h"
    "${CMAKE_SOURCE_DIR}/src/httpserver/qhttpserverrouterrule_p.h"
    DESTINATION "${HTTP_PRIV_ROOT}/private")

# ── Qt5SslServer ──────────────────────────────────────────────────────────────
if(QT_HAVE_SSL)
    add_library(Qt5SslServer
        src/sslserver/qsslserver.cpp
        src/sslserver/qsslserver.h
        src/sslserver/qsslserver_p.h
        src/sslserver/qtsslserverglobal.h
    )

    if(BUILD_SHARED_LIBS)
        target_compile_definitions(Qt5SslServer PRIVATE QT_BUILD_SSLSERVER_LIB)
    else()
        target_compile_definitions(Qt5SslServer PUBLIC QT_STATIC)
    endif()

    target_include_directories(Qt5SslServer
        PUBLIC
            # <QtSslServer/qsslserver.h> forwarding headers live here
            $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
            # Direct source-level includes inside the library
            $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src/sslserver>
        PRIVATE
            # Resolves <private/qsslserver_p.h>
            "${SSL_PRIV_ROOT}"
    )

    target_link_libraries(Qt5SslServer
        PUBLIC
            Qt5::Core
            Qt5::Network
    )

    set_target_properties(Qt5SslServer PROPERTIES
        OUTPUT_NAME "Qt5SslServer"
        DEBUG_POSTFIX "d"
        VERSION ${PROJECT_VERSION}
        SOVERSION 5
    )
endif()

# ── Qt5HttpServer ─────────────────────────────────────────────────────────────
set(HTTPSERVER_SOURCES
    src/3rdparty/http-parser/http_parser.c
    src/httpserver/qabstracthttpserver.cpp
    src/httpserver/qhttpserver.cpp
    src/httpserver/qhttpserverliterals.cpp
    src/httpserver/qhttpserverrequest.cpp
    src/httpserver/qhttpserverresponder.cpp
    src/httpserver/qhttpserverresponse.cpp
    src/httpserver/qhttpserverrouter.cpp
    src/httpserver/qhttpserverrouterrule.cpp
)

set(HTTPSERVER_HEADERS
    src/httpserver/qthttpserverglobal.h
    src/httpserver/qabstracthttpserver.h
    src/httpserver/qabstracthttpserver_p.h
    src/httpserver/qhttpserver.h
    src/httpserver/qhttpserver_p.h
    src/httpserver/qhttpserverliterals_p.h
    src/httpserver/qhttpserverrequest.h
    src/httpserver/qhttpserverrequest_p.h
    src/httpserver/qhttpserverresponder.h
    src/httpserver/qhttpserverresponder_p.h
    src/httpserver/qhttpserverresponse.h
    src/httpserver/qhttpserverresponse_p.h
    src/httpserver/qhttpserverrouter.h
    src/httpserver/qhttpserverrouter_p.h
    src/httpserver/qhttpserverrouterrule.h
    src/httpserver/qhttpserverrouterrule_p.h
    src/httpserver/qhttpserverrouterviewtraits.h
    src/httpserver/qhttpserverviewtraits.h
    src/httpserver/qhttpserverviewtraits_impl.h
)

if(Qt5Concurrent_FOUND)
    list(APPEND HTTPSERVER_SOURCES src/httpserver/qhttpserverfutureresponse.cpp)
    list(APPEND HTTPSERVER_HEADERS src/httpserver/qhttpserverfutureresponse.h)
endif()

add_library(Qt5HttpServer
    ${HTTPSERVER_SOURCES}
    ${HTTPSERVER_HEADERS}
)

if(BUILD_SHARED_LIBS)
    target_compile_definitions(Qt5HttpServer PRIVATE QT_BUILD_HTTPSERVER_LIB)
else()
    target_compile_definitions(Qt5HttpServer PUBLIC QT_STATIC)
endif()

target_include_directories(Qt5HttpServer
    PUBLIC
        # <QtHttpServer/qabstracthttpserver.h> forwarding headers live here
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        # Direct source includes (e.g. qthttpserverglobal.h)
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src/httpserver>
    PRIVATE
        # Resolves <private/qabstracthttpserver_p.h> etc.
        "${HTTP_PRIV_ROOT}"
        # Bundled http-parser
        "${CMAKE_SOURCE_DIR}/src/3rdparty/http-parser"
)

target_link_libraries(Qt5HttpServer
    PUBLIC
        Qt5::Core
        Qt5::Network
    PRIVATE
        # Needed for <private/qobject_p.h> used in *_p.h headers
        Qt5::CorePrivate
)

if(Qt5Concurrent_FOUND)
    target_link_libraries(Qt5HttpServer PUBLIC Qt5::Concurrent)
endif()

if(Qt5WebSockets_FOUND)
    target_link_libraries(Qt5HttpServer PUBLIC Qt5::WebSockets)
    # The source guards WebSocket code with this macro
    target_compile_definitions(Qt5HttpServer PRIVATE QT_WEBSOCKETS_LIB)
endif()

if(QT_HAVE_SSL AND TARGET Qt5SslServer)
    target_link_libraries(Qt5HttpServer PUBLIC Qt5SslServer)
    # Expose SslServer headers so consumers see <QtSslServer/...>
    target_include_directories(Qt5HttpServer PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/src/sslserver>
    )
endif()

set_target_properties(Qt5HttpServer PROPERTIES
    OUTPUT_NAME "Qt5HttpServer"
    DEBUG_POSTFIX "d"
    VERSION ${PROJECT_VERSION}
    SOVERSION 5
)
