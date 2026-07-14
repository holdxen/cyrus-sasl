add_library(plugin_common STATIC
    plugin_common.c
    plugin_common.h
)
target_include_directories(plugin_common PUBLIC
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}
)
if(WIN32)
    target_link_libraries(plugin_common PRIVATE ${LIB_SOCKET})
endif()

add_library(crypto_compat STATIC
    crypto-compat.c
    crypto-compat.h
)
target_include_directories(crypto_compat PUBLIC
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}
    ${OPENSSL_INCLUDE_DIR}
)
target_link_libraries(crypto_compat PRIVATE OpenSSL::Crypto)
