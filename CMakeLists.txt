cmake_minimum_required(VERSION 3.16)
project(ladybird-gtk4 C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(PkgConfig)
pkg_check_modules(gtk4 REQUIRED IMPORTED_TARGET gtk4>=4.11.3)
pkg_check_modules(libadwaita REQUIRED IMPORTED_TARGET libadwaita-1>=1.4)

find_program(GLIB_COMPILE_RESOURCES NAMES glib-compile-resources REQUIRED)

#find_program(BLUEPRINT_COMPILER NAMES blueprint-compiler REQUIRED)

include(ExternalProject)
ExternalProject_Add(blueprint-compiler
    GIT_REPOSITORY https://gitlab.gnome.org/jwestman/blueprint-compiler.git
    GIT_TAG v0.10.0
    GIT_SHALLOW ON
    CONFIGURE_COMMAND
        meson setup ../blueprint-compiler
            # This is a hack to pass the proper prefix directory to Meson.
            -D prefix=$<TARGET_PROPERTY:blueprint-compiler,_EP_INSTALL_DIR>
    BUILD_COMMAND meson compile
    TEST_COMMAND meson test
    TEST_EXCLUDE_FROM_MAIN ON
    INSTALL_COMMAND meson install
    EXCLUDE_FROM_ALL
)

set(BLUEPRINT_COMPILER "$<TARGET_PROPERTY:blueprint-compiler,_EP_INSTALL_DIR>/bin/blueprint-compiler")

set(LAGOM_SOURCE_DIR "${SERENITY_SOURCE_DIR}/Meta/Lagom")
set(LAGOM_BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/Lagom")

include("${SERENITY_SOURCE_DIR}/Meta/CMake/lagom_compile_options.cmake")

set(BUILD_LAGOM ON CACHE INTERNAL "")
set(ENABLE_LAGOM_LIBWEB ON CACHE INTERNAL "")
# We set EXCLUDE_FROM_ALL to make sure that only required Lagom libraries are built
add_subdirectory("${LAGOM_SOURCE_DIR}" "${LAGOM_BINARY_DIR}" EXCLUDE_FROM_ALL)
# FIXME: Why doesn't Lagom do this?
include_directories(
    "${SERENITY_SOURCE_DIR}"
    "${SERENITY_SOURCE_DIR}/Userland/Libraries"
    "${LAGOM_BINARY_DIR}"
    "${LAGOM_BINARY_DIR}/Userland/Libraries"
)
add_compile_definitions(AK_DONT_REPLACE_STD)

foreach(prefix_dir IN LISTS CMAKE_PREFIX_PATH)
    list(APPEND GI_TYPELIB_PATH "${prefix_dir}/lib64/girepository-1.0")
endforeach()
list(JOIN GI_TYPELIB_PATH ":" GI_TYPELIB_PATH_ENV)

set(RESOURCE_FILES)
function(add_blueprint source_file ui_file)
    add_custom_command(
        OUTPUT "${ui_file}"
        COMMAND "${CMAKE_COMMAND}"
        ARGS
            -E env GI_TYPELIB_PATH="${GI_TYPELIB_PATH_ENV}"
            "${BLUEPRINT_COMPILER}"
            compile
            --output "${CMAKE_CURRENT_BINARY_DIR}/${ui_file}"
            "${CMAKE_CURRENT_SOURCE_DIR}/${source_file}"
        DEPENDS blueprint-compiler "${source_file}"
    )
    list(APPEND RESOURCE_FILES "${ui_file}")
    set(RESOURCE_FILES ${RESOURCE_FILES} PARENT_SCOPE)
endfunction()

add_blueprint(window.blp window.ui)

set(SOURCE_GRESOURCE_FILE "${CMAKE_CURRENT_SOURCE_DIR}/gresources.xml")
set(GENERATED_GRESOURCE_FILE "${CMAKE_CURRENT_BINARY_DIR}/ladybird-resources.c")

add_custom_command(
    OUTPUT "${GENERATED_GRESOURCE_FILE}"
    COMMAND "${GLIB_COMPILE_RESOURCES}"
    ARGS
        --target "${GENERATED_GRESOURCE_FILE}"
        --generate-source
        --sourcedir "${CMAKE_CURRENT_SOURCE_DIR}"
        --sourcedir "${CMAKE_CURRENT_BINARY_DIR}"
        "${SOURCE_GRESOURCE_FILE}"
    DEPENDS "${SOURCE_GRESOURCE_FILE}" ${RESOURCE_FILES}
)

set(SOURCES
    main.cpp
    Window.cpp
    WebContentView.cpp
    ViewImpl.cpp
    "${GENERATED_GRESOURCE_FILE}"
)

add_executable(ladybird ${SOURCES})
target_link_libraries(ladybird PRIVATE PkgConfig::gtk4 PkgConfig::libadwaita Lagom::Core Lagom::WebView Lagom::Gfx Lagom::Web Lagom::IPC)

include_directories(
    "${SERENITY_SOURCE_DIR}/Userland/Services"
)

set(WEBCONTENT_SOURCE_DIR "${SERENITY_SOURCE_DIR}/Userland/Services/WebContent")
set(WEBCONTENT_SOURCES
    "${WEBCONTENT_SOURCE_DIR}/ConnectionFromClient.cpp"
    "${WEBCONTENT_SOURCE_DIR}/ConsoleGlobalEnvironmentExtensions.cpp"
    "${WEBCONTENT_SOURCE_DIR}/PageHost.cpp"
    "${WEBCONTENT_SOURCE_DIR}/WebContentConsoleClient.cpp"
    "${WEBCONTENT_SOURCE_DIR}/WebDriverConnection.cpp"
    WebContentMain.cpp
)
add_executable(WebContent ${WEBCONTENT_SOURCES})
target_link_libraries(WebContent PRIVATE Lagom::Audio Lagom::Core Lagom::FileSystem Lagom::Gfx Lagom::IPC Lagom::JS Lagom::Main Lagom::Web Lagom::WebSocket)
