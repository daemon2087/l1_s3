cmake_minimum_required(VERSION 3.10)

## Determine project version: prefer Git tag, fall back to default. The
## resulting value is stored in the cache variable `PROJECT_VERSION_OVERRIDE`
## so it can still be overridden by `-DPROJECT_VERSION_OVERRIDE=...`.
set(DEFAULT_PROJECT_VERSION "0.0.1 beta")

add_executable(helloworld_cli main.cpp)
add_library(helloworld lib.cpp)
add_executable(test_version test_version.cpp)

    # Try to detect the most recent tag from Git (requires Git and that the
    # source directory is a git checkout with tags). We strip a leading 'v'
    # if present (tags like "v1.0.4" become "1.0.4").
    find_package(Git QUIET)
    if(GIT_FOUND)
        execute_process(
            COMMAND ${GIT_EXECUTABLE} describe --tags --abbrev=0
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            RESULT_VARIABLE _git_describe_result
            OUTPUT_VARIABLE _git_describe_out
            OUTPUT_STRIP_TRAILING_WHITESPACE
            ERROR_QUIET
        )
    endif()

    if(DEFINED _git_describe_result AND _git_describe_result EQUAL 0 AND _git_describe_out)
        # Remove optional leading 'v'
        string(REGEX REPLACE "^v" "" _project_version "${_git_describe_out}")
    else()
        set(_project_version "${DEFAULT_PROJECT_VERSION}")
    endif()

    set(PROJECT_VERSION_OVERRIDE "${_project_version}" CACHE STRING "Override project version")

    project(helloworld VERSION ${PROJECT_VERSION_OVERRIDE} LANGUAGES C CXX)
    configure_file(version.h.in version.h)
    # Now that project() has been called and C++ is configured, find Boost
    find_package(Boost COMPONENTS unit_test_framework REQUIRED)
set_target_properties(helloworld_cli helloworld test_version PROPERTIES
    CXX_STANDARD 14
    CXX_STANDARD_REQUIRED ON
)
target_include_directories(helloworld
    PRIVATE "${CMAKE_BINARY_DIR}"
)
set_target_properties(test_version PROPERTIES
    COMPILE_DEFINITIONS BOOST_TEST_DYN_LINK
    INCLUDE_DIRECTORIES ${Boost_INCLUDE_DIR}
)

target_link_libraries(helloworld_cli PRIVATE
    helloworld
)
target_link_libraries(test_version
    ${Boost_LIBRARIES}
    helloworld
)

if (MSVC)
    target_compile_options(helloworld_cli PRIVATE
        /W4
    )
    target_compile_options(helloworld PRIVATE
        /W4
    )
    target_compile_options(test_version PRIVATE
        /W4
    )
else ()
    target_compile_options(helloworld_cli PRIVATE
        -Wall -Wextra -pedantic -Werror
    )
    target_compile_options(helloworld PRIVATE
        -Wall -Wextra -pedantic -Werror
    )
    target_compile_options(test_version PRIVATE
        -Wall -Wextra -pedantic -Werror
    )
endif()

install(TARGETS helloworld_cli RUNTIME DESTINATION bin)

set(CPACK_GENERATOR DEB)

set(CPACK_PACKAGE_VERSION_MAJOR "${PROJECT_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${PROJECT_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${PROJECT_VERSION_PATCH}")
set(CPACK_PACKAGE_CONTACT pdiv@tpu.ru)

# Устанавливаем генератор CPack в зависимости от ОС
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    set(CPACK_GENERATOR "DEB")
elseif(CMAKE_SYSTEM_NAME MATCHES "Darwin")
    set(CPACK_GENERATOR "TGZ")
endif()

include(CPack)

enable_testing()

add_test(test_version test_version)
