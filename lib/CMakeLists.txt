set(LIBSASL_SOURCES
    auxprop.c
    canonusr.c
    checkpw.c
    client.c
    common.c
    config.c
    external.c
    saslutil.c
    server.c
    seterror.c
)

if(WIN32)
    list(APPEND LIBSASL_SOURCES windlopen.c)
else()
    list(APPEND LIBSASL_SOURCES dlopen.c)
endif()

if(NOT HAVE_GETSUBOPT)
    list(APPEND LIBSASL_SOURCES getsubopt.c)
endif()

if(NOT HAVE_SNPRINTF OR NOT HAVE_VSNPRINTF)
    list(APPEND LIBSASL_SOURCES snprintf.c)
endif()

if(NOT HAVE_GETADDRINFO)
    list(APPEND LIBSASL_SOURCES getaddrinfo.c)
endif()

if(NOT HAVE_GETNAMEINFO)
    list(APPEND LIBSASL_SOURCES getnameinfo.c)
endif()

set(LIBSASL_PRIVATE_HEADERS
    saslint.h
    staticopen.h
)

if(SASL_SHARED)
    add_library(sasl2 SHARED ${LIBSASL_SOURCES} ${LIBSASL_PRIVATE_HEADERS})
else()
    add_library(sasl2 STATIC ${LIBSASL_SOURCES} ${LIBSASL_PRIVATE_HEADERS})
endif()

set_target_properties(sasl2 PROPERTIES
    OUTPUT_NAME "sasl2"
    VERSION "${SASL_VERSION_MAJOR}.${SASL_VERSION_MINOR}.${SASL_VERSION_STEP}"
    SOVERSION "${SASL_VERSION_MAJOR}"
)

target_include_directories(sasl2 PUBLIC
    $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

target_include_directories(sasl2 PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/plugins
    ${CMAKE_SOURCE_DIR}/sasldb
    ${CMAKE_SOURCE_DIR}/common
)

target_compile_definitions(sasl2 PRIVATE
    LIBSASL_EXPORTS=1
    CONFIGDIR="${SASL_CONFIGDIR}"
    PLUGINDIR="${SASL_PLUGINDIR}"
)

target_link_libraries(sasl2 PRIVATE
    plugin_common
    OpenSSL::Crypto
)

if(WIN32)
    target_link_libraries(sasl2 PRIVATE ${LIB_SOCKET})
else()
    if(SASL_DL_LIB)
        target_link_libraries(sasl2 PRIVATE ${SASL_DL_LIB})
    endif()
    if(LIB_SOCKET)
        target_link_libraries(sasl2 PRIVATE ${LIB_SOCKET})
    endif()
    if(HAVE_PTHREAD)
        target_link_libraries(sasl2 PRIVATE Threads::Threads)
    endif()
endif()

install(TARGETS sasl2
    EXPORT cyrus-sasl-targets
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
)
