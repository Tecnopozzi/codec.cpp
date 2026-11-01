# libttsbackbone: the pinned llama.cpp (+ its bundled ggml, static) wrapped
# into one shared library that exports only llama_* and the common_sampler
# C++ API; every ggml_* symbol stays local (see SetupTtsBackbone.cmake).
# A5 / MSVC: WINDOWS_EXPORT_ALL_SYMBOLS does NOT re-export WHOLEARCHIVE
# members — use an explicit .def generated from llama.lib symbols.

cmake_minimum_required(VERSION 3.16)
project(ttsbackbone C CXX)

set(LLAMA_DIR "" CACHE PATH "llama.cpp source directory")
set(TTSBACKBONE_OUTPUT_DIR "${CMAKE_BINARY_DIR}" CACHE PATH "where libttsbackbone goes")

add_subdirectory("${LLAMA_DIR}" llama EXCLUDE_FROM_ALL)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/ttsbackbone.c" "")
add_library(ttsbackbone SHARED "${CMAKE_CURRENT_BINARY_DIR}/ttsbackbone.c")
set_target_properties(ttsbackbone PROPERTIES
    LINKER_LANGUAGE CXX
    LIBRARY_OUTPUT_DIRECTORY "${TTSBACKBONE_OUTPUT_DIR}"
    RUNTIME_OUTPUT_DIRECTORY "${TTSBACKBONE_OUTPUT_DIR}"
    ARCHIVE_OUTPUT_DIRECTORY "${TTSBACKBONE_OUTPUT_DIR}"
    # Multi-config generators (VS) append $<CONFIG>; keep wheel install path flat.
    LIBRARY_OUTPUT_DIRECTORY_RELEASE "${TTSBACKBONE_OUTPUT_DIR}"
    RUNTIME_OUTPUT_DIRECTORY_RELEASE "${TTSBACKBONE_OUTPUT_DIR}"
    ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${TTSBACKBONE_OUTPUT_DIR}"
    LIBRARY_OUTPUT_DIRECTORY_DEBUG "${TTSBACKBONE_OUTPUT_DIR}"
    RUNTIME_OUTPUT_DIRECTORY_DEBUG "${TTSBACKBONE_OUTPUT_DIR}"
    ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${TTSBACKBONE_OUTPUT_DIR}")
target_link_libraries(ttsbackbone PRIVATE llama llama-common)

if(APPLE)
    target_link_options(ttsbackbone PRIVATE
        "LINKER:-force_load,$<TARGET_FILE:llama>"
        "LINKER:-force_load,$<TARGET_FILE:llama-common>")
    set(_exports "${CMAKE_CURRENT_BINARY_DIR}/ttsbackbone.exports")
    file(WRITE "${_exports}" "_llama_*\n__Z*common_sampler*\n")
    target_link_options(ttsbackbone PRIVATE "LINKER:-exported_symbols_list,${_exports}")
elseif(MSVC)
    set(_def "${CMAKE_CURRENT_SOURCE_DIR}/ttsbackbone_exports.def")
    target_link_options(ttsbackbone PRIVATE
        "/WHOLEARCHIVE:$<TARGET_FILE:llama>"
        "/WHOLEARCHIVE:$<TARGET_FILE:llama-common>"
        "/DEF:${_def}")
    # Ensure def participates in rebuilds
    set_target_properties(ttsbackbone PROPERTIES LINK_DEPENDS "${_def}")
else()
    target_link_options(ttsbackbone PRIVATE
        "LINKER:--whole-archive" "$<TARGET_FILE:llama>" "$<TARGET_FILE:llama-common>" "LINKER:--no-whole-archive")
    set(_exports "${CMAKE_CURRENT_BINARY_DIR}/ttsbackbone.map")
    file(WRITE "${_exports}"
        "{\n  global:\n    llama_*;\n    extern \"C++\" {\n      common_sampler*;\n    };\n  local:\n    *;\n};\n")
    target_link_options(ttsbackbone PRIVATE "LINKER:--version-script,${_exports}")
endif()
