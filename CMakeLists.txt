cmake_minimum_required(VERSION 2.8)
project(xmrig)

option(WITH_LIBCPUID        "Enable libcpuid support" ON)
option(WITH_HWLOC           "Enable hwloc support" ON)
option(WITH_CN_LITE         "Enable CryptoNight-Lite algorithms family" ON)
option(WITH_CN_HEAVY        "Enable CryptoNight-Heavy algorithms family" ON)
option(WITH_CN_PICO         "Enable CryptoNight-Pico algorithm" ON)
option(WITH_CN_GPU          "Enable CryptoNight-GPU algorithm" ON)
option(WITH_RANDOMX         "Enable RandomX algorithms family" ON)
option(WITH_ARGON2          "Enable Argon2 algorithms family" ON)
option(WITH_HTTP            "Enable HTTP protocol support (client/server)" ON)
option(WITH_DEBUG_LOG       "Enable debug log output" OFF)
option(WITH_TLS             "Enable OpenSSL support" ON)
option(WITH_ASM             "Enable ASM PoW implementations" ON)
option(WITH_EMBEDDED_CONFIG "Enable internal embedded JSON config" OFF)
option(WITH_OPENCL          "Enable OpenCL backend" ON)
option(WITH_STRICT_CACHE    "Enable strict checks for OpenCL cache" ON)
option(WITH_INTERLEAVE_DEBUG_LOG "Enable debug log for threads interleave" OFF)

option(BUILD_STATIC         "Build static binary" OFF)
option(ARM_TARGET           "Force use specific ARM target 8 or 7" 0)
option(HWLOC_DEBUG          "Enable hwloc debug helpers and log" OFF)


set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")


include (CheckIncludeFile)
include (cmake/cpu.cmake)
include (src/base/base.cmake)
include (src/backend/backend.cmake)


set(HEADERS
    "${HEADERS_BASE}"
    "${HEADERS_BASE_HTTP}"
    "${HEADERS_BACKEND}"
    src/App.h
    src/core/config/Config_default.h
    src/core/config/Config_platform.h
    src/core/config/Config.h
    src/core/config/ConfigTransform.h
    src/core/config/usage.h
    src/core/Controller.h
    src/core/Miner.h
    src/net/interfaces/IJobResultListener.h
    src/net/JobResult.h
    src/net/JobResults.h
    src/net/Network.h
    src/net/NetworkState.h
    src/net/strategies/DonateStrategy.h
    src/Summary.h
    src/version.h
   )

set(HEADERS_CRYPTO
    src/crypto/cn/asm/CryptonightR_template.h
    src/crypto/cn/c_blake256.h
    src/crypto/cn/c_groestl.h
    src/crypto/cn/c_jh.h
    src/crypto/cn/c_skein.h
    src/crypto/cn/CnAlgo.h
    src/crypto/cn/CnCtx.h
    src/crypto/cn/CnHash.h
    src/crypto/cn/CryptoNight_monero.h
    src/crypto/cn/CryptoNight_test.h
    src/crypto/cn/CryptoNight.h
    src/crypto/cn/groestl_tables.h
    src/crypto/cn/hash.h
    src/crypto/cn/skein_port.h
    src/crypto/cn/soft_aes.h
    src/crypto/common/Algorithm.h
    src/crypto/common/Coin.h
    src/crypto/common/keccak.h
    src/crypto/common/Nonce.h
    src/crypto/common/portable/mm_malloc.h
    src/crypto/common/VirtualMemory.h
   )

if (XMRIG_ARM)
    set(HEADERS_CRYPTO "${HEADERS_CRYPTO}" src/crypto/cn/CryptoNight_arm.h)
else()
    set(HEADERS_CRYPTO "${HEADERS_CRYPTO}" src/crypto/cn/CryptoNight_x86.h)
endif()

set(SOURCES
    "${SOURCES_BASE}"
    "${SOURCES_BASE_HTTP}"
    "${SOURCES_BACKEND}"
    src/App.cpp
    src/core/config/Config.cpp
    src/core/config/ConfigTransform.cpp
    src/core/Controller.cpp
    src/core/Miner.cpp
    src/core/Benchmark.cpp
    src/net/JobResults.cpp
    src/net/Network.cpp
    src/net/NetworkState.cpp
    src/net/strategies/DonateStrategy.cpp
    src/Summary.cpp
    src/xmrig.cpp
   )

set(SOURCES_CRYPTO
    src/crypto/cn/c_blake256.c
    src/crypto/cn/c_groestl.c
    src/crypto/cn/c_jh.c
    src/crypto/cn/c_skein.c
    src/crypto/cn/CnCtx.cpp
    src/crypto/cn/CnHash.cpp
    src/crypto/common/Algorithm.cpp
    src/crypto/common/Coin.cpp
    src/crypto/common/keccak.cpp
    src/crypto/common/Nonce.cpp
    src/crypto/common/VirtualMemory.cpp
   )

if (WIN32)
    set(SOURCES_OS
        "${SOURCES_OS}"
        res/app.rc
        src/App_win.cpp
        src/crypto/common/VirtualMemory_win.cpp
        )

    add_definitions(/DWIN32)
    set(EXTRA_LIBS ws2_32 psapi iphlpapi userenv)
elseif (APPLE)
    set(SOURCES_OS
        "${SOURCES_OS}"
        src/App_unix.cpp
        src/crypto/common/VirtualMemory_unix.cpp
        )
else()
    set(SOURCES_OS
        "${SOURCES_OS}"
        src/App_unix.cpp
        src/crypto/common/VirtualMemory_unix.cpp
        )

    if (CMAKE_SYSTEM_NAME STREQUAL FreeBSD)
        set(EXTRA_LIBS kvm pthread)
    else()
        set(EXTRA_LIBS pthread rt dl)
    endif()
endif()

if (CMAKE_SYSTEM_NAME MATCHES "Linux" OR CMAKE_SYSTEM_NAME MATCHES "Android")
    EXECUTE_PROCESS(COMMAND uname -o COMMAND tr -d '\n' OUTPUT_VARIABLE OPERATING_SYSTEM)
    if (OPERATING_SYSTEM MATCHES "Android")
        set(EXTRA_LIBS ${EXTRA_LIBS} log)
    endif()
endif()

add_definitions(/D__STDC_FORMAT_MACROS)
add_definitions(/DUNICODE)

find_package(UV REQUIRED)

include(cmake/flags.cmake)
include(cmake/randomx.cmake)
include(cmake/argon2.cmake)
include(cmake/OpenSSL.cmake)
include(cmake/asm.cmake)
include(cmake/cn-gpu.cmake)

if (WITH_CN_LITE)
    add_definitions(/DXMRIG_ALGO_CN_LITE)
endif()

if (WITH_CN_HEAVY)
    add_definitions(/DXMRIG_ALGO_CN_HEAVY)
endif()

if (WITH_CN_PICO)
    add_definitions(/DXMRIG_ALGO_CN_PICO)
endif()

if (WITH_EMBEDDED_CONFIG)
    add_definitions(/DXMRIG_FEATURE_EMBEDDED_CONFIG)
endif()

include_directories(src)
include_directories(src/3rdparty)
include_directories(${UV_INCLUDE_DIR})

if (BUILD_STATIC)
    set(CMAKE_EXE_LINKER_FLAGS " -static")
endif()

if (WITH_DEBUG_LOG)
    add_definitions(/DAPP_DEBUG)
endif()

add_executable(${CMAKE_PROJECT_NAME} ${HEADERS} ${SOURCES} ${SOURCES_OS} ${SOURCES_CPUID} ${HEADERS_CRYPTO} ${SOURCES_CRYPTO} ${SOURCES_SYSLOG} ${TLS_SOURCES} ${XMRIG_ASM_SOURCES} ${CN_GPU_SOURCES})
target_link_libraries(${CMAKE_PROJECT_NAME} ${XMRIG_ASM_LIBRARY} ${OPENSSL_LIBRARIES} ${UV_LIBRARIES} ${EXTRA_LIBS} ${CPUID_LIB} ${ARGON2_LIBRARY})
