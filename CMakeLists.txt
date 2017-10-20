cmake_minimum_required (VERSION 2.8.11)
project (libADLMIDI)

#===========================================================================================
# Strip garbage
if(APPLE)
    set(LINK_FLAGS_RELEASE  "${LINK_FLAGS_RELEASE} -dead_strip")
ELSEIF(NOT MSVC)
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -Os -s -fdata-sections -ffunction-sections -Wl,--gc-sections -Wl,-s")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Os -s -fdata-sections -ffunction-sections -Wl,--gc-sections -Wl,-s")
    set(LINK_FLAGS_RELEASE  "${LINK_FLAGS_RELEASE} -Wl,--gc-sections -Wl,-s")
ENDIF()

# Global optimization flags
IF(NOT MSVC)
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -fno-omit-frame-pointer")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fno-omit-frame-pointer")
ENDIF()

if (CMAKE_BUILD_TYPE EQUAL "RELEASE")
    add_definitions(-DNDEBUG)
ENDIF()

# Disable bogus MSVC warnings
if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
ENDIF()

# -fPIC thing
IF(NOT WIN32)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
ENDIF()
#===========================================================================================

option(WITH_MIDIPLAY        "Build also demo MIDI player" OFF)
option(WITH_OLD_UTILS       "Build also old utilities" OFF)
option(WITH_EMBEDDED_BANKS  "Use embedded banks" ON)
option(USE_DOSBOX_EMULATOR  "Use DosBox emulator" OFF)

option(libADLMIDI_STATIC   "Build static library of libADLMIDI" ON)
option(libADLMIDI_SHARED   "Build shared library of libADLMIDI" OFF)



set(libADLMIDI_INSTALLS)

include_directories(${libADLMIDI_SOURCE_DIR}/include)
include_directories(${libADLMIDI_SOURCE_DIR}/src/)
link_directories(${libADLMIDI_BINARY_DIR}/)

if(WITH_OLD_UTILS)
    # Old utility to dump AdLib bank files
    add_executable(adldumpbank
        ${libADLMIDI_SOURCE_DIR}/utils/dumpbank/dumpbank.cpp
    )
    list(APPEND libADLMIDI_INSTALLS adldumpbank)

    # Old utility to dump AIL bank files
    add_executable(adldumpmiles
        ${libADLMIDI_SOURCE_DIR}/utils/dumpmiles/dumpmiles.cpp
    )
    list(APPEND libADLMIDI_INSTALLS adldumpmiles)
endif()

if(WITH_EMBEDDED_BANKS)
    add_executable(gen_adldata
        ${libADLMIDI_SOURCE_DIR}/utils/gen_adldata/gen_adldata.cc
        ${libADLMIDI_SOURCE_DIR}/utils/gen_adldata/progs_cache.cpp
        ${libADLMIDI_SOURCE_DIR}/utils/gen_adldata/measurer.cpp
        ${libADLMIDI_SOURCE_DIR}/utils/gen_adldata/ini/ini_processing.cpp
        ${libADLMIDI_SOURCE_DIR}/src/nukedopl3.c
        ${libADLMIDI_SOURCE_DIR}/src/dbopl.cpp
    )
    target_link_libraries(gen_adldata pthread)

    add_custom_target (
        gen-adldata-run
        COMMAND ${libADLMIDI_BINARY_DIR}/gen_adldata ${libADLMIDI_SOURCE_DIR}/src/adldata.cpp
        WORKING_DIRECTORY ${libADLMIDI_SOURCE_DIR}
    )
    add_dependencies(gen-adldata-run gen_adldata)
endif()


set(libADLMIDI_SOURCES)

list(APPEND libADLMIDI_SOURCES
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi.cpp
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_load.cpp
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_midiplay.cpp
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_opl3.cpp
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_private.cpp
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_mus2mid.c
    ${libADLMIDI_SOURCE_DIR}/src/adlmidi_xmi2mid.c
)

if(USE_DOSBOX_EMULATOR)
    add_definitions(-DADLMIDI_USE_DOSBOX_OPL)
    list(APPEND libADLMIDI_SOURCES
        ${libADLMIDI_SOURCE_DIR}/src/dbopl.cpp
    )
else()
    list(APPEND libADLMIDI_SOURCES
        ${libADLMIDI_SOURCE_DIR}/src/nukedopl3.c
    )
endif()

if(WITH_EMBEDDED_BANKS)
    list(APPEND libADLMIDI_SOURCES
        ${libADLMIDI_SOURCE_DIR}/src/adldata.cpp
    )
else()
    add_definitions(-DDISABLE_EMBEDDED_BANKS)
endif()

if(libADLMIDI_STATIC)
    add_library(ADLMIDI_static STATIC ${libADLMIDI_SOURCES})
    set_target_properties(ADLMIDI_static PROPERTIES OUTPUT_NAME ADLMIDI)
    target_include_directories(ADLMIDI_static PUBLIC ${SDLMixerX_SOURCE_DIR}/include)
    list(APPEND libADLMIDI_INSTALLS ADLMIDI_static)
    if(WITH_EMBEDDED_BANKS)
        add_dependencies(ADLMIDI_static gen-adldata-run)
    endif()
endif()

if(libADLMIDI_SHARED)
    add_library(ADLMIDI_shared SHARED ${libADLMIDI_SOURCES})
    set_target_properties(ADLMIDI_shared PROPERTIES OUTPUT_NAME ADLMIDI)
    target_include_directories(ADLMIDI_shared PUBLIC ${SDLMixerX_SOURCE_DIR}/include)
    list(APPEND libADLMIDI_INSTALLS ADLMIDI_shared)
    if(WITH_EMBEDDED_BANKS)
        add_dependencies(ADLMIDI_shared gen-adldata-run)
    endif()
endif()

if(WITH_MIDIPLAY)
    find_library(SDL2_LIBRARY SDL2 REQUIRED)
    include_directories(${SDL2_INCLUDE_DIR})
    message("Found ${SDL2_LIBRARY}")

    add_executable(adlmidiplay
        ${libADLMIDI_SOURCE_DIR}/utils/midiplay/adlmidiplay.cpp
        ${libADLMIDI_SOURCE_DIR}/utils/midiplay/wave_writer.c
    )

    if(WIN32)
        target_link_libraries(adlmidiplay ADLMIDI ${SDL2_LIBRARY} pthread)
    else()
        target_link_libraries(adlmidiplay ADLMIDI ${SDL2_LIBRARY} pthread dl)
    endif()

    if(libADLMIDI_SHARED)
        add_dependencies(adlmidiplay ADLMIDI_shared)
        # ========= WIP =========
        # set_target_properties(adlmidiplay PROPERTIES COMPILE_FLAGS "-Wl,-rpath='$$ORIGIN/../lib'")
    else()
        if(NOT libADLMIDI_STATIC)
            message(FATAL_ERROR "libADLMIDI is required to be built!")
        endif()
        add_dependencies(adlmidiplay ADLMIDI_static)
    endif()

    list(APPEND libADLMIDI_INSTALLS adlmidiplay)
endif()

install(TARGETS ${libADLMIDI_INSTALLS}
        RUNTIME DESTINATION "bin"
        LIBRARY DESTINATION "lib"
        ARCHIVE DESTINATION "lib"
        INCLUDES DESTINATION "include")

install(FILES
        include/adlmidi.h
        DESTINATION include/)

message("==== libADLMIDI options ====")
message("WITH_MIDIPLAY          = ${WITH_MIDIPLAY}")
message("WITH_EMBEDDED_BANKS    = ${WITH_EMBEDDED_BANKS}")
message("USE_DOSBOX_EMULATOR    = ${USE_DOSBOX_EMULATOR}")
message("libADLMIDI_STATIC      = ${libADLMIDI_STATIC}")
message("libADLMIDI_SHARED      = ${libADLMIDI_SHARED}")

