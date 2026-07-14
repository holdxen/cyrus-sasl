set(SASLDB_SOURCES
    allockey.c
    sasldb.h
    ${SASL_DB_BACKEND}
)

add_library(sasldb STATIC ${SASLDB_SOURCES})

target_include_directories(sasldb PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}
)

target_compile_definitions(sasldb PRIVATE LIBSASL_EXPORTS=1)

if(SASL_DB_LIB)
    target_link_libraries(sasldb PUBLIC ${SASL_DB_LIB})
endif()
