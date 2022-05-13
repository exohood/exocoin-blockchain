cmake_minimum_required(VERSION 2.8)

set(Exohood
        exoblocks)


project(${Exohood})

set(CMAKE_CXX_STANDARD 11)

if (WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wa,-mbig-obj -O3")
endif()


if (NOT EXOHOOD_DIR)
    set(EXOHOOD_DIR ~/EXOHOOD)
endif()

message(STATUS EXOHOOD_DIR ": ${EXOHOOD_DIR}")

if (NOT EXOHOOD_SOURCE_DIR)
    set(EXOHOOD_SOURCE_DIR ${EXOHOOD_DIR}
          CACHE PATH "Path to the root directory for EXOHOOD")
endif()

if (NOT EXOHOOD_BUILD_DIR)
    # set location of EXOHOOD build tree
    set(EXOHOOD_BUILD_DIR ${EXOHOOD_SOURCE_DIR}/build/release/
          CACHE PATH "Path to the build directory for EXOHOOD")
endif()

set(MY_CMAKE_DIR "${CMAKE_CURRENT_LIST_DIR}/cmake"
        CACHE PATH "The path to the cmake directory of the current project")

list(APPEND CMAKE_MODULE_PATH "${MY_CMAKE_DIR}")

set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${EXOHOOD_BUILD_DIR}"
        CACHE PATH "Add EXOHOOD directory for library searching")

include(MyUtils)

find_package(EXOHOOD)

# find boost
find_package(Boost COMPONENTS
        system
        filesystem
        thread
        date_time
        chrono
        regex
        serialization
        program_options
        date_time
        REQUIRED)

#info https://github.com/arsenm/sanitizers-cmake
find_package(Sanitizers)

if(APPLE)
    include_directories(/usr/local/opt/openssl/include)
    link_directories(/usr/local/opt/openssl/lib)
endif()


if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin" OR WIN32)
    add_library(unbound STATIC IMPORTED)
    set_property(TARGET unbound PROPERTY IMPORTED_LOCATION ${EXOHOOD_BUILD_DIR}/external/unbound/libunbound.a)
endif()



# include boost headers
include_directories(${Boost_INCLUDE_DIRS})

# include EXOHOOD
include_directories(${EXOHOOD_SOURCE_DIR}/build)

include_directories("ext/mstch/include")
include_directories("ext/mstch/include/src")
include_directories("ext/crow")

# add ext/ subfolder
add_subdirectory(ext/)

# add src/ subfolder
add_subdirectory(src/)


set(SOURCE_FILES
        main.cpp)

#ADD_CUSTOM_TARGET(driver DEPENDS src/templates/index.html)

add_executable(${Exohood}
        ${SOURCE_FILES})

add_sanitizers(${Exohood})

create_git_version()

configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates ${CMAKE_CURRENT_BINARY_DIR}/templates)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/css ${CMAKE_CURRENT_BINARY_DIR}/templates/css)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/partials ${CMAKE_CURRENT_BINARY_DIR}/templates/partials)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/js ${CMAKE_CURRENT_BINARY_DIR}/templates/js)
configure_files(${CMAKE_CURRENT_SOURCE_DIR}/src/templates/imgs ${CMAKE_CURRENT_BINARY_DIR}/templates/imgs)

set(LIBRARIES
        myxrm
        myext
        mstch
        wallet
        blockchain_db
        cryptonote_core
        cryptonote_protocol
        cryptonote_basic
        multisig
        daemonizer
        cncrypto
        blocks
        lmdb
        ringct
        ringct_basic
        device
        common
        mnemonics
        easylogging
        checkpoints
        version
        epee
        sodium
        ${Boost_LIBRARIES}
        pthread
        unbound
        curl
        crypto
        ssl
	ed25519-donna)
if(APPLE)
    set(LIBRARIES ${LIBRARIES} "-framework IOKit -framework Foundation")
else()
    set(LIBRARIES ${LIBRARIES} atomic)
endif()

if (NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin" AND NOT WIN32)
    set(LIBRARIES ${LIBRARIES} unwind)
endif()

if (WIN32)
    set(LIBRARIES ${LIBRARIES}
            wsock32
            ntdll
            ws2_32
            Iphlpapi
            )
else()
    set(LIBRARIES ${LIBRARIES} dl)
endif()

find_package(HIDAPI)
set(LIBRARIES ${LIBRARIES} ${HIDAPI_LIBRARIES})

target_link_libraries(${Exohood} ${LIBRARIES})
