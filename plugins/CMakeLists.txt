set(PLUGIN_COMMON_INCLUDES
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/lib
    ${CMAKE_SOURCE_DIR}/sasldb
    ${CMAKE_SOURCE_DIR}/common
    ${CMAKE_BINARY_DIR}
)

function(sasl_add_plugin name)
    cmake_parse_arguments(PARSE_ARGV 1 ARG "" "" "SOURCES;LIBS;DEFINITIONS")
    add_library(${name} MODULE ${ARG_SOURCES})
    target_include_directories(${name} PRIVATE ${PLUGIN_COMMON_INCLUDES})
    target_compile_definitions(${name} PRIVATE LIBSASL_EXPORTS=1 ${ARG_DEFINITIONS})
    target_link_libraries(${name} PRIVATE plugin_common ${ARG_LIBS})
    if(WIN32)
        target_link_libraries(${name} PRIVATE ${LIB_SOCKET})
    endif()
    set_target_properties(${name} PROPERTIES
        PREFIX ""
        OUTPUT_NAME "${name}"
    )
    install(TARGETS ${name}
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}/sasl2"
        RUNTIME DESTINATION "${CMAKE_INSTALL_LIBDIR}/sasl2"
    )
endfunction()

function(sasl_generate_init mech type)
    set(init_file "${CMAKE_CURRENT_BINARY_DIR}/${mech}_init.c")
    if("${type}" STREQUAL "mech")
        file(WRITE "${init_file}" "
#include <config.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#ifndef macintosh
#include <sys/stat.h>
#endif
#include <fcntl.h>
#include <assert.h>
#include <sasl.h>
#include <saslplug.h>
#include <saslutil.h>
#include \"plugin_common.h\"

#ifdef WIN32
BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call) {
    case DLL_PROCESS_ATTACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
#endif

SASL_CLIENT_PLUG_INIT(${mech})
SASL_SERVER_PLUG_INIT(${mech})
")
    elseif("${type}" STREQUAL "auxprop")
        file(WRITE "${init_file}" "
#include <config.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#ifndef macintosh
#include <sys/stat.h>
#endif
#include <fcntl.h>
#include <assert.h>
#include <sasl.h>
#include <saslplug.h>
#include <saslutil.h>
#include \"plugin_common.h\"

#ifdef WIN32
BOOL APIENTRY DllMain(HANDLE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call) {
    case DLL_PROCESS_ATTACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
#endif

SASL_AUXPROP_PLUG_INIT(${mech})
")
    endif()
    set(${mech}_INIT_FILE "${init_file}" PARENT_SCOPE)
endfunction()

if(SASL_ENABLE_ANON)
    sasl_generate_init(anonymous mech)
    sasl_add_plugin(saslANONYMOUS
        SOURCES anonymous.c "${anonymous_INIT_FILE}"
    )
endif()

if(SASL_ENABLE_PLAIN)
    sasl_generate_init(plain mech)
    sasl_add_plugin(saslPLAIN
        SOURCES plain.c "${plain_INIT_FILE}"
    )
endif()

if(SASL_ENABLE_SCRAM)
    sasl_generate_init(scram mech)
    sasl_add_plugin(saslSCRAM
        SOURCES scram.c "${scram_INIT_FILE}"
        LIBS OpenSSL::Crypto
        DEFINITIONS HAVE_SHA512=1
    )
endif()

if(SASL_ENABLE_OTP)
    sasl_generate_init(otp mech)
    sasl_add_plugin(saslOTP
        SOURCES otp.c "${otp_INIT_FILE}" otp.h
        LIBS OpenSSL::Crypto
    )
endif()

if(SASL_ENABLE_SRP)
    sasl_generate_init(srp mech)
    sasl_add_plugin(saslSRP
        SOURCES srp.c "${srp_INIT_FILE}"
        LIBS OpenSSL::Crypto crypto_compat
    )
endif()

if(SASL_ENABLE_PASSDSS)
    sasl_generate_init(passdss mech)
    sasl_add_plugin(saslPASSDSS
        SOURCES passdss.c "${passdss_INIT_FILE}"
        LIBS OpenSSL::Crypto crypto_compat
    )
endif()

if(SASL_ENABLE_OPAQUE)
    find_library(OPAQUE_LIB opaque)
    find_library(SODIUM_LIB sodium)
    check_include_file("sodium.h" HAVE_SODIUM_H)

    if(OPAQUE_LIB)
        set(HAVE_LIBOPAQUE 1)
        set(OPAQUE_LIBS ${OPAQUE_LIB})
        set(OPAQUE_EXTRA_SRCS "")
    elseif(SODIUM_LIB AND HAVE_SODIUM_H)
        set(HAVE_LIBSODIUM 1)
        set(OPAQUE_LIBS ${SODIUM_LIB})
        set(OPAQUE_EXTRA_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/opaque_impl.c")
        check_symbol_exists(crypto_kdf_hkdf_sha512_expand "sodium/crypto_kdf.h" HAVE_SODIUM_HKDF)
        if(NOT HAVE_SODIUM_HKDF)
            list(APPEND OPAQUE_EXTRA_SRCS "${CMAKE_CURRENT_SOURCE_DIR}/kdf_hkdf_sha512.c")
        endif()
    else()
        message(WARNING "Neither libopaque nor libsodium found -- OPAQUE will be disabled")
        set(SASL_ENABLE_OPAQUE OFF)
    endif()

    if(SASL_ENABLE_OPAQUE)
        sasl_generate_init(opaque mech)
        sasl_add_plugin(saslOPAQUE
            SOURCES opaque.c "${opaque_INIT_FILE}" ${OPAQUE_EXTRA_SRCS} opaque.h
            LIBS ${OPAQUE_LIBS} crypto_compat
        )
    endif()
endif()

if(SASL_WITH_GSSAPI AND HAVE_GSSAPI)
    sasl_generate_init(gssapiv2 mech)
    sasl_add_plugin(saslGSSAPI
        SOURCES gssapi.c "${gssapiv2_INIT_FILE}"
        LIBS ${KRB5_LIBRARIES}
        DEFINITIONS HAVE_GSSAPI=1
    )
    target_include_directories(saslGSSAPI PRIVATE ${KRB5_INCLUDE_DIRS})

    if(SASL_ENABLE_GS2)
        sasl_generate_init(gs2 mech)
        sasl_add_plugin(saslGS2
            SOURCES gs2.c "${gs2_INIT_FILE}" gs2_token.c gs2_token.h
            LIBS ${KRB5_LIBRARIES}
            DEFINITIONS HAVE_GSSAPI=1
        )
        target_include_directories(saslGS2 PRIVATE ${KRB5_INCLUDE_DIRS})
    endif()
endif()

if(SASL_ENABLE_SASLDB)
    sasl_generate_init(sasldb auxprop)
    sasl_add_plugin(saslSASLDB
        SOURCES sasldb.c "${sasldb_INIT_FILE}"
        LIBS sasldb
    )
    target_include_directories(saslSASLDB PRIVATE
        ${CMAKE_SOURCE_DIR}/sasldb
    )
endif()

if(SASL_WITH_SQL)
    set(SQL_LIBS "")
    set(SQL_DEFS "")

    if(SASL_WITH_MYSQL)
        find_library(MYSQL_LIB mysqlclient)
        check_include_file("mysql/mysql.h" HAVE_MYSQL_H)
        if(MYSQL_LIB AND HAVE_MYSQL_H)
            set(HAVE_MYSQL 1)
            list(APPEND SQL_LIBS ${MYSQL_LIB})
            list(APPEND SQL_DEFS HAVE_MYSQL=1)
        endif()
    endif()

    if(SASL_WITH_PGSQL)
        find_library(PGSQL_LIB pq)
        check_include_file("libpq-fe.h" HAVE_PGSQL_H)
        if(PGSQL_LIB)
            set(HAVE_PGSQL 1)
            list(APPEND SQL_LIBS ${PGSQL_LIB})
            list(APPEND SQL_DEFS HAVE_PGSQL=1)
        endif()
    endif()

    if(SASL_WITH_SQLITE)
        find_library(SQLITE_LIB sqlite)
        check_include_file("sqlite.h" HAVE_SQLITE_H)
        if(SQLITE_LIB AND HAVE_SQLITE_H)
            set(HAVE_SQLITE 1)
            list(APPEND SQL_LIBS ${SQLITE_LIB})
            list(APPEND SQL_DEFS HAVE_SQLITE=1)
        endif()
    endif()

    if(SASL_WITH_SQLITE3)
        find_library(SQLITE3_LIB sqlite3)
        check_include_file("sqlite3.h" HAVE_SQLITE3_H)
        if(SQLITE3_LIB AND HAVE_SQLITE3_H)
            set(HAVE_SQLITE3 1)
            list(APPEND SQL_LIBS ${SQLITE3_LIB})
            list(APPEND SQL_DEFS HAVE_SQLITE3=1)
        endif()
    endif()

    if(SQL_LIBS)
        sasl_generate_init(sql auxprop)
        sasl_add_plugin(saslSQL
            SOURCES sql.c "${sql_INIT_FILE}"
            LIBS ${SQL_LIBS}
            DEFINITIONS ${SQL_DEFS}
        )
    else()
        message(WARNING "SQL enabled but no database library found")
    endif()
endif()

if(SASL_ENABLE_LDAPDB AND HAVE_LDAP)
    sasl_generate_init(ldapdb auxprop)
    set(ldapdb_init_file "${CMAKE_CURRENT_BINARY_DIR}/ldapdb_init.c")
    file(APPEND "${ldapdb_init_file}" "SASL_CANONUSER_PLUG_INIT(ldapdb)\n")
    sasl_add_plugin(saslLDAPDB
        SOURCES ldapdb.c "${ldapdb_init_file}"
        LIBS ${LDAP_LIB} ${LBER_LIB} OpenSSL::Crypto
    )
endif()
