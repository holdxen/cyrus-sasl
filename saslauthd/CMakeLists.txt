set(SASLAUTHD_SOURCES
    mechanisms.c
    mechanisms.h
    globals.h
    auth_getpwent.c
    auth_getpwent.h
    auth_rimap.c
    auth_rimap.h
    auth_shadow.c
    auth_shadow.h
    auth_httpform.c
    auth_httpform.h
    cache.c
    cache.h
    cfile.c
    cfile.h
    utils.c
    utils.h
    ipc_unix.c
    saslauthd-main.c
    saslauthd-main.h
)

set(SASLAUTHD_INCLUDES
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}
    ${CMAKE_SOURCE_DIR}/common
    ${CMAKE_BINARY_DIR}/common
)

set(SASLAUTHD_LIBS OpenSSL::Crypto)

if(HAVE_PAM)
    list(APPEND SASLAUTHD_SOURCES auth_pam.c auth_pam.h)
    list(APPEND SASLAUTHD_LIBS ${PAM_LIB})
endif()

if(HAVE_GSSAPI AND KRB5_FOUND)
    list(APPEND SASLAUTHD_SOURCES auth_krb5.c auth_krb5.h)
    list(APPEND SASLAUTHD_LIBS ${KRB5_LIBRARIES})
endif()

if(HAVE_LDAP)
    list(APPEND SASLAUTHD_SOURCES auth_ldap.c auth_ldap.h lak.c lak.h)
    list(APPEND SASLAUTHD_LIBS ${LDAP_LIB} ${LBER_LIB})
endif()

if(AUTH_SASLDB)
    list(APPEND SASLAUTHD_SOURCES auth_sasldb.c auth_sasldb.h)
    list(APPEND SASLAUTHD_LIBS sasldb)
endif()

if(LIB_SOCKET)
    list(APPEND SASLAUTHD_LIBS ${LIB_SOCKET})
endif()

if(HAVE_PTHREAD)
    list(APPEND SASLAUTHD_LIBS Threads::Threads)
endif()

list(APPEND SASLAUTHD_LIBS crypto_compat)

add_executable(saslauthd ${SASLAUTHD_SOURCES})
target_include_directories(saslauthd PRIVATE ${SASLAUTHD_INCLUDES})
target_compile_definitions(saslauthd PRIVATE
    SASLAUTHD_CONF_FILE_DEFAULT="${CMAKE_INSTALL_FULL_SYSCONFDIR}/saslauthd.conf"
)
target_link_libraries(saslauthd PRIVATE ${SASLAUTHD_LIBS})
install(TARGETS saslauthd RUNTIME DESTINATION "${CMAKE_INSTALL_SBINDIR}")

add_executable(testsaslauthd testsaslauthd.c utils.c)
target_include_directories(testsaslauthd PRIVATE ${SASLAUTHD_INCLUDES})
if(LIB_SOCKET)
    target_link_libraries(testsaslauthd PRIVATE ${LIB_SOCKET})
endif()
install(TARGETS testsaslauthd RUNTIME DESTINATION "${CMAKE_INSTALL_SBINDIR}")
