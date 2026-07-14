set(TEST_INCLUDES
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_BINARY_DIR}
)

add_executable(t_gssapi_cli t_common.c t_gssapi_cli.c)
target_include_directories(t_gssapi_cli PRIVATE ${TEST_INCLUDES})
target_link_libraries(t_gssapi_cli PRIVATE sasl2)
if(LIB_SOCKET)
    target_link_libraries(t_gssapi_cli PRIVATE ${LIB_SOCKET})
endif()

add_executable(t_gssapi_srv t_common.c t_gssapi_srv.c)
target_include_directories(t_gssapi_srv PRIVATE ${TEST_INCLUDES})
target_link_libraries(t_gssapi_srv PRIVATE sasl2)
if(LIB_SOCKET)
    target_link_libraries(t_gssapi_srv PRIVATE ${LIB_SOCKET})
endif()

add_test(NAME t_gssapi COMMAND "${CMAKE_CURRENT_SOURCE_DIR}/runtests.py")
