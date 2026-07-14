set(SASL_PUBLIC_HEADERS
    sasl.h
    saslplug.h
    saslutil.h
    prop.h
)

install(FILES ${SASL_PUBLIC_HEADERS}
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/sasl"
)
