cmake_minimum_required(VERSION 3.20)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

set(CMAKE_INSTALL_BINDIR "." CACHE STRING "Binary dir for install")

set(QT_DEFAULT_MAJOR_VERSION 6 CACHE STRING "Qt version to use (5 or 6), defaults to 6")
find_package(Qt${QT_DEFAULT_MAJOR_VERSION} REQUIRED COMPONENTS Core Gui Network PrintSupport Sql Svg Widgets WebChannel WebEngineWidgets LinguistTools)
find_package(Qt${QT_DEFAULT_MAJOR_VERSION} OPTIONAL_COMPONENTS Core5Compat)

if ((QT_DEFAULT_MAJOR_VERSION GREATER 5))
    qt_standard_project_setup()
else()
    set(CMAKE_AUTOMOC ON)
    set(CMAKE_AUTORCC ON)
    set(CMAKE_AUTOUIC ON)
endif()

# Application icon on Windows
set(VX_APP_ICON_RC_WIN data/core/icons/vnote.rc)

# The MACOSX_BUNDLE_ICON_FILE variable is added to the Info.plist
# generated by CMake. This variable contains the .icns file name,
# without the path.
set(MACOSX_BUNDLE_ICON_FILE vnote.icns)
# And the following tells CMake where to find and install the file itself.
set(VX_APP_ICON_MACOS data/core/icons/vnote.icns)
set_source_files_properties(${VX_APP_ICON_MACOS} PROPERTIES
    MACOSX_PACKAGE_LOCATION "Resources")

# Translations
set(VX_TS_FILES data/core/translations/vnote_zh_CN.ts
    data/core/translations/vnote_ja.ts)
if((QT_DEFAULT_MAJOR_VERSION EQUAL 6))
    if((Qt6Widgets_VERSION VERSION_GREATER_EQUAL 6.7.0))
        qt_add_lupdate(TS_FILES ${VX_TS_FILES}
            SOURCE_TARGETS vnote)
    else()
        qt_add_lupdate(vnote TS_FILES ${VX_TS_FILES})
    endif()
endif()
# Generate .qm files from .ts files (lrelease)
set_source_files_properties(${VX_TS_FILES} PROPERTIES
    OUTPUT_LOCATION "${CMAKE_CURRENT_BINARY_DIR}/translations")
qt_add_translation(VX_QM_FILES ${VX_TS_FILES})
add_custom_target(lrelease DEPENDS ${VX_QM_FILES})

# TODO: VTextEdit translations
list(APPEND VX_QM_FILES
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qdialogbuttonbox_zh_CN.qm
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qtbase_ja.qm
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qtbase_zh_CN.qm
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qtv_ja.qm
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qtv_zh_CN.qm
    ${CMAKE_CURRENT_LIST_DIR}/data/core/translations/qwebengine_zh_CN.qm
)

# Resources
set(VX_RESOURCE_FILES data/core/core.qrc)
set(VX_EXTRA_RESOURCE_FILES_RCC ${CMAKE_CURRENT_BINARY_DIR}/vnote_extra.rcc)
qt_add_binary_resources(VX_EXTRA_RESOURCE data/extra/extra.qrc DESTINATION ${VX_EXTRA_RESOURCE_FILES_RCC} OPTIONS -compress 9)

add_executable(vnote WIN32 MACOSX_BUNDLE
    application.cpp application.h
    commandlineoptions.cpp commandlineoptions.h
    fakeaccessible.cpp fakeaccessible.h
    main.cpp
    ${VX_APP_ICON_RC_WIN} ${VX_APP_ICON_MACOS} ${VX_RESOURCE_FILES}
)

add_dependencies(vnote VX_EXTRA_RESOURCE)

set(VX_LIBS_FOLDER ../libs)
target_include_directories(vnote PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
)

find_program(GOLD_LINKER "ld.gold")
if (NOT ${GOLD_LINKER} STREQUAL GOLD_LINKER-NOTFOUND)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=gold")
endif()

target_compile_definitions(vnote PRIVATE
    QT_MESSAGELOGCONTEXT
)

add_subdirectory(core)
add_subdirectory(export)
add_subdirectory(imagehost)
add_subdirectory(search)
add_subdirectory(snippet)
add_subdirectory(task)
add_subdirectory(unitedentry)
add_subdirectory(utils)
add_subdirectory(widgets)

target_link_libraries(vnote PRIVATE
    Qt::Core
    Qt::Gui
    Qt::Network
    Qt::PrintSupport
    Qt::Sql
    Qt::Svg
    Qt::WebChannel
    Qt::WebEngineWidgets
    Qt::Widgets
    VTextEdit
    qhotkey
)

if((QT_DEFAULT_MAJOR_VERSION GREATER 5))
    target_link_libraries(vnote PRIVATE
        Qt::Core5Compat
    )
endif()

# Copy the qt.conf on Windows
if(WIN32)
    add_custom_command(TARGET vnote POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${PROJECT_SOURCE_DIR}/package/qt.conf" $<TARGET_FILE_DIR:vnote>)
endif()

# Installation
if (WIN32)
    install(TARGETS vnote RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
    install(FILES "${PROJECT_SOURCE_DIR}/package/qt.conf" DESTINATION ${CMAKE_INSTALL_BINDIR})
    install(FILES ${VX_EXTRA_RESOURCE_FILES_RCC} DESTINATION ${CMAKE_INSTALL_BINDIR})
    install(FILES ${VX_QM_FILES} DESTINATION "${CMAKE_INSTALL_BINDIR}/translations" OPTIONAL)
elseif(APPLE)
    set(CMAKE_MACOSX_RPATH ON)

    # TODO: declare install for macOS if necessary. For packing, we will manually copy files into
    # the src/vnote.app bundle.

    # The generated Info.plist will be overridden.
    set_target_properties(vnote
        PROPERTIES
        OUTPUT_NAME "${PROJECT_NAME}"
        MACOSX_BUNDLE_BUNDLE_NAME "${PROJECT_NAME}"
        MACOSX_BUNDLE_INFO_STRING "${PROJECT_DESCRIPTION}"
        MACOSX_BUNDLE_GUI_IDENTIFIER "fun.vnote.vnote"
        MACOSX_BUNDLE_LONG_VERSION_STRING "${PROJECT_VERSION}"
        MACOSX_BUNDLE_SHORT_VERSION_STRING "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}"
        MACOSX_BUNDLE_BUNDLE_VERSION "${PROJECT_VERSION}"
        MACOSX_BUNDLE_COPYRIGHT "Distributed under LGPL-3.0 license. Copyright (c) 2024 app.vnote.fun"
    )
else()
    install(TARGETS vnote
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
    install(FILES ${VX_EXTRA_RESOURCE_FILES_RCC} DESTINATION ${CMAKE_INSTALL_DATADIR})
    install(FILES ${VX_QM_FILES} DESTINATION "${CMAKE_INSTALL_DATADIR}/translations" OPTIONAL)

    set(desktop.path applications)
    set(desktop.files data/core/vnote.desktop)
    set(icon16.path icons/hicolor/16x16/apps)
    set(icon16.files data/core/logo/16x16/vnote.png)
    set(icon32.path icons/hicolor/32x32/apps)
    set(icon32.files data/core/logo/32x32/vnote.png)
    set(icon48.path icons/hicolor/48x48/apps)
    set(icon48.files data/core/logo/48x48/vnote.png)
    set(icon64.path icons/hicolor/64x64/apps)
    set(icon64.files data/core/logo/64x64/vnote.png)
    set(icon128.path icons/hicolor/128x128/apps)
    set(icon128.files data/core/logo/128x128/vnote.png)
    set(icon256.path icons/hicolor/256x256/apps)
    set(icon256.files data/core/logo/256x256/vnote.png)
    set(iconsvg.path icons/hicolor/scalable/apps)
    set(iconsvg.files data/core/logo/vnote.svg)
    foreach(item IN ITEMS desktop icon16 icon32 icon48 icon64 icon128 icon256 iconsvg)
        install(FILES ${CMAKE_CURRENT_LIST_DIR}/${${item}.files}
            DESTINATION ${CMAKE_INSTALL_DATADIR}/${${item}.path}
            PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ)
    endforeach()
endif()

include(${CMAKE_CURRENT_LIST_DIR}/Packaging.cmake)
