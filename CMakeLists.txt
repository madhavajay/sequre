cmake_minimum_required(VERSION 3.14)
project(Sequre)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pedantic -Wall -Wno-return-type-c-linkage")
set(CMAKE_CXX_FLAGS_DEBUG "-g -fno-limit-debug-info")
set(CMAKE_CXX_FLAGS_RELEASE "-O3")
include_directories(.)

set(CPM_DOWNLOAD_VERSION 0.32.3)
set(CPM_DOWNLOAD_LOCATION "${CMAKE_BINARY_DIR}/cmake/CPM_${CPM_DOWNLOAD_VERSION}.cmake")
if(NOT (EXISTS ${CPM_DOWNLOAD_LOCATION}))
    file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/cmake")
    # Prefer a locally built copy from the Codon tree to avoid network fetches.
    set(CPM_FALLBACK "${CMAKE_SOURCE_DIR}/../codon/build/cmake/CPM_${CPM_DOWNLOAD_VERSION}.cmake")
    if(EXISTS ${CPM_FALLBACK})
        message(STATUS "Using bundled CPM.cmake from ${CPM_FALLBACK}")
        file(COPY ${CPM_FALLBACK} DESTINATION "${CMAKE_BINARY_DIR}/cmake")
    else()
        message(STATUS "Downloading CPM.cmake...")
        file(DOWNLOAD https://github.com/TheLartians/CPM.cmake/releases/download/v${CPM_DOWNLOAD_VERSION}/CPM.cmake ${CPM_DOWNLOAD_LOCATION})
    endif()
endif()
include(${CPM_DOWNLOAD_LOCATION})

set(SYQURE_SKIP_XZ ON) # default to skipping xz to avoid network fetches
if(DEFINED ENV{SYQURE_SKIP_XZ} AND "$ENV{SYQURE_SKIP_XZ}" STREQUAL "1")
    # stay skipped
elseif(DEFINED ENV{ALLOW_XZ_FETCH})
    set(SYQURE_SKIP_XZ OFF)
elseif(DEFINED ENV{XZ_SOURCE_DIR} AND EXISTS "$ENV{XZ_SOURCE_DIR}")
    message(STATUS "Using pre-downloaded xz from $ENV{XZ_SOURCE_DIR}")
    set(XZ_SOURCE_DIR_OVERRIDE "$ENV{XZ_SOURCE_DIR}")
    set(SYQURE_SKIP_XZ OFF)
endif()

if(NOT SYQURE_SKIP_XZ)
    CPMAddPackage(
        NAME xz
        GITHUB_REPOSITORY "xz-mirror/xz"
        VERSION 5.2.5
        GIT_TAG e7da44d5151e21f153925781ad29334ae0786101
        SOURCE_DIR ${XZ_SOURCE_DIR_OVERRIDE}
        EXCLUDE_FROM_ALL YES
        OPTIONS "BUILD_SHARED_LIBS OFF"
                "CMAKE_POSITION_INDEPENDENT_CODE ON")
    if(xz_ADDED)
        set_target_properties(xz PROPERTIES EXCLUDE_FROM_ALL ON)
        set_target_properties(xzdec PROPERTIES EXCLUDE_FROM_ALL ON)
    endif()
endif()

set(SEQURE_FILES
    ir/analysis/consecutive_matmul.cpp
    ir/analysis/consecutive_matmul.h
    ir/analysis/dead_code.cpp
    ir/analysis/dead_code.h
    ir/helpers/utils.cpp
    ir/helpers/utils.h
    ir/helpers/bet.cpp
    ir/helpers/bet.h
    ir/obsolete/mpc.cpp
    ir/obsolete/mpc.h
    ir/mhe.cpp
    ir/mhe.h
    ir/expr.cpp
    ir/expr.h
    ir/debugger.cpp
    ir/debugger.h
    ir/sequre.cpp
    ir/sequre.h)
add_library(sequre SHARED ${SEQURE_FILES})

find_package(LLVM REQUIRED CONFIG)
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})

if(NOT CODON_PATH)
    set(CODON_PATH "$ENV{SEQURE_CODON_PATH}/install")
endif()
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "${CODON_PATH}/lib/codon/plugins/sequre" CACHE PATH "Use the existing Codon installation" FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

target_include_directories(sequre PRIVATE "${CODON_PATH}/include" ${LLVM_INCLUDE_DIRS})
target_link_directories(sequre PRIVATE "${CODON_PATH}/lib/codon")
target_link_libraries(sequre PRIVATE codonrt codonc)

install(TARGETS sequre DESTINATION build)
install(FILES plugin.toml DESTINATION .)
install(DIRECTORY ${CMAKE_SOURCE_DIR}/stdlib DESTINATION .)
