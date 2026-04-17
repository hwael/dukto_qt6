cmake_minimum_required(VERSION 3.16 FATAL_ERROR)
If (APPLE)
    set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64")
endif()
project(dukto)

#FIXME: updater no longer works
#OPTION(USE_UPDATER "Add updater for application" OFF)
OPTION(USE_SINGLE_APP "Allow only one instance" OFF)
OPTION(USE_NOTIFY_LIBNOTIFY "Use libnotify for notifications (Linux only)" OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

if(USE_UPDATER)
    add_definitions("-DUPDATER")
endif()
if(USE_SINGLE_APP AND NOT ANDROID)
    add_definitions("-DSINGLE_APP")
endif()
if(USE_NOTIFY_LIBNOTIFY AND UNIX AND NOT APPLE AND NOT ANDROID)
    add_definitions("-DNOTIFY_LIBNOTIFY")
endif()

if(ANDROID)
    set(ANDROID_ABI ${CMAKE_ANDROID_ARCH_ABI})
    add_definitions("-DMOBILE_APP")
else()
    add_definitions("-DDESKTOP_APP")
endif()

if(QT_CMAKE_ROOT)
    get_filename_component(QT_CMAKE_ROOT "${QT_CMAKE_ROOT}" ABSOLUTE)
    if(NOT IS_DIRECTORY "${QT_CMAKE_ROOT}")
        message(FATAL_ERROR "The directory specified by QT_CMAKE_ROOT was not found")
    endif()
    list(PREPEND CMAKE_PREFIX_PATH "${QT_CMAKE_ROOT}")
endif()

set(QT_COMPONENTS Core Gui Network Qml QuickWidgets)
if(UNIX AND NOT APPLE AND NOT ANDROID)
    list(APPEND QT_COMPONENTS DBus)
endif()
# Use a single Qt 6.6+ baseline across all platforms, including Android packaging
# features that rely on QTP0002.
find_package(Qt6 6.6 COMPONENTS ${QT_COMPONENTS} REQUIRED)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions("-DQT_NO_DEBUG_OUTPUT")
endif()

add_definitions("-DUNICODE")

include_directories(${CMAKE_CURRENT_SOURCE_DIR})

set(DUKTO_HDR
    buddylistitemmodel.h
    destinationbuddy.h
    duktoprotocol.h
    duktowindow.h
    guibehind.h
    ipaddressitemmodel.h
    miniwebserver.h
    network/buddymessage.h
    network/filedata.h
    network/messenger.h
    network/receiver.h
    network/sender.h
    peer.h
    platform.h
    recentlistitemmodel.h
    settings.h
    systemtray.h
    theme.h
)

set(DUKTO_SRC
    buddylistitemmodel.cpp
    destinationbuddy.cpp
    duktoprotocol.cpp
    duktowindow.cpp
    guibehind.cpp
    ipaddressitemmodel.cpp
    main.cpp
    miniwebserver.cpp
    network/buddymessage.cpp
    network/filedata.cpp
    network/messenger.cpp
    network/receiver.cpp
    network/sender.cpp
    platform.cpp
    recentlistitemmodel.cpp
    settings.cpp
    systemtray.cpp
    theme.cpp
)

set(DUKTO_RESOURCES
    qml.qrc
    qml/common/common.qrc
)
list(APPEND DUKTO_RESOURCES "qml/new/main.qrc")

if(USE_UPDATER)
    list(APPEND DUKTO_SRC updateschecker.cpp)
    list(APPEND DUKTO_HDR updateschecker.h)
endif()

if(WIN32)
    list(APPEND DUKTO_SRC ecwin7.cpp)
    list(APPEND DUKTO_HDR ecwin7.h)
    list(APPEND DUKTO_RESOURCES dukto.rc)
endif()

if(ANDROID)
    list(APPEND DUKTO_SRC androidutils.cpp)
    list(APPEND DUKTO_HDR androidutils.h)
endif()

if(USE_SINGLE_APP AND NOT ANDROID)
    set(QAPPLICATION_CLASS "QApplication")
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/modules/SingleApplication)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/modules/SingleApplication)
endif()

if(USE_NOTIFY_LIBNOTIFY AND UNIX AND NOT APPLE AND NOT ANDROID)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(LIBNOTIFY REQUIRED libnotify)
    include_directories(${LIBNOTIFY_INCLUDE_DIRS})
    list(APPEND DUKTO_LIBS ${LIBNOTIFY_LIBRARIES})
endif()

if(ANDROID)
    qt_add_executable(${PROJECT_NAME}
                      ${DUKTO_HDR}
                      ${DUKTO_SRC}
                      ${DUKTO_RESOURCES})
else()
    add_executable(${PROJECT_NAME}
                   ${DUKTO_HDR}
                   ${DUKTO_SRC}
                   ${DUKTO_RESOURCES})
endif(ANDROID)

if(USE_SINGLE_APP AND NOT ANDROID)
    add_dependencies(${PROJECT_NAME} SingleApplication)
    link_directories(${CMAKE_CURRENT_BINARY_DIR}/modules/SingleApplication)
    target_link_libraries(${PROJECT_NAME} PRIVATE SingleApplication)
endif()

set(QT_LIBRARIES "")
foreach(temp ${QT_COMPONENTS})
    list(APPEND QT_LIBRARIES "Qt6::${temp}")
endforeach()
target_link_libraries(${PROJECT_NAME} PRIVATE ${DUKTO_LIBS} ${QT_LIBRARIES})

if(WIN32)
    set_target_properties(${PROJECT_NAME} PROPERTIES WIN32_EXECUTABLE True)
    target_link_libraries(${PROJECT_NAME} PRIVATE Ws2_32 ole32 user32)
endif()

If (APPLE)
    set(MACOSX_BUNDLE_ICON_FILE dukto.icns)
    set_source_files_properties("dukto.icns" PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
    target_sources(${PROJECT_NAME} PRIVATE ${MACOSX_BUNDLE_ICON_FILE})
    set_property(TARGET ${PROJECT_NAME} PROPERTY MACOSX_BUNDLE 1)
    find_library(CORE_SERVICES CoreServices REQUIRED)
    target_link_options(${PROJECT_NAME} PRIVATE "${CORE_SERVICES}/CoreServices.tbd")
endif()

if(ANDROID)
    # QTP0002 enables modern Android multi-ABI package source handling.
    qt6_policy(SET QTP0002 NEW)
    # get version
    file(READ "${CMAKE_CURRENT_SOURCE_DIR}/version.h" ver)
    string(REGEX MATCH "#define VERSION_MAJOR ([0-9]+)" _ ${ver})
    set(ver_major ${CMAKE_MATCH_1})
    string(REGEX MATCH "#define VERSION_MINOR ([0-9]+)" _ ${ver})
    set(ver_minor ${CMAKE_MATCH_1})
    string(REGEX MATCH "#define VERSION_PATCH ([0-9]+)" _ ${ver})
    set(ver_patch ${CMAKE_MATCH_1})
    # get target abis
    if(QT_ANDROID_BUILD_ALL_ABIS)
        set(target_abis ${QT_DEFAULT_ANDROID_ABIS})
    else()
        set(target_abis ${QT_ANDROID_ABIS})
    endif()
    # calculate abi code
    set(abi_code 0)
    foreach(abi IN ITEMS ${target_abis})
        if(abi STREQUAL "armeabi-v7a")
            math(EXPR abi_code "${abi_code} | 1")
        elseif(abi STREQUAL "arm64-v8a")
            math(EXPR abi_code "${abi_code} | 2")
        elseif(abi STREQUAL "x86_64")
            math(EXPR abi_code "${abi_code} | 4")
        elseif(abi STREQUAL "x86")
            math(EXPR abi_code "${abi_code} | 8")
        else()
            message(FATAL_ERROR "Unknown ABI: ${abi}")
        endif()
    endforeach()
    # calculate version code
    set(ver_code "${ver_major}")
    string(LENGTH "${ver_minor}" len)
    if(len EQUAL 1)
        set(ver_code "${ver_code}0${ver_minor}")
    else()
        set(ver_code "${ver_code}${ver_minor}")
    endif()
    string(LENGTH "${ver_patch}" len)
    if(len EQUAL 1)
        set(ver_code "${ver_code}0${ver_patch}")
    else()
        set(ver_code "${ver_code}${ver_patch}")
    endif()
    string(LENGTH "${abi_code}" len)
    if(len EQUAL 1)
        set(ver_code "${ver_code}260${abi_code}")
    else()
        set(ver_code "${ver_code}26${abi_code}")
    endif()

    set_target_properties(dukto PROPERTIES
        QT_ANDROID_MIN_SDK_VERSION 26
        QT_ANDROID_VERSION_NAME "${ver_major}.${ver_minor}.${ver_patch}"
        QT_ANDROID_VERSION_CODE "${ver_code}"
        QT_ANDROID_PACKAGE_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/android/qt6"
    )
endif(ANDROID)

if(UNIX AND NOT APPLE AND NOT ANDROID)
    install(TARGETS ${PROJECT_NAME}
            DESTINATION bin)
    install(FILES dukto.png
            DESTINATION share/pixmaps/)
    install(FILES dukto.desktop
            DESTINATION share/applications/)
endif()
