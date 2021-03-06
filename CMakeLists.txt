cmake_minimum_required(VERSION 2.8)

project(faf_uid CXX C)

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" UID_COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" UID_COMPILER_SUPPORTS_CXX0X)
if(UID_COMPILER_SUPPORTS_CXX11)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(UID_COMPILER_SUPPORTS_CXX0X)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
  message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

if(NOT DEFINED JSONCPP_LIBRARIES)
  if(NOT PKGCONFIG_FOUND)
    find_package(PkgConfig REQUIRED)
  endif()
  pkg_search_module(JSONCPP REQUIRED jsoncpp)
endif()

include_directories(${JSONCPP_INCLUDE_DIRS})

if(NOT DEFINED CRYPTOPP_LIBRARIES)
  if(NOT PKGCONFIG_FOUND)
    find_package(PkgConfig REQUIRED)
  endif()
  pkg_search_module(CRYPTOPP REQUIRED cryptopp)
endif()

include_directories(${CRYPTOPP_INCLUDE_DIRS})

if (WIN32)
  if(NOT PKGCONFIG_FOUND)
    find_package(PkgConfig REQUIRED)
  endif()
  pkg_search_module(ICU REQUIRED icu-uc)
  include_directories(${ICU_INCLUDE_DIRS})
  set(machine_info_src machine_info_win.cpp)
  set(machine_info_libs
    ${ICU_LIBRARIES}
    wbemuuid.lib
      )
else()
  set(machine_info_src machine_info_linux.cpp)
endif()

if(NOT DEFINED UID_PUBKEY_BYTES)
  message(WARNING "You need to set UID_PUBKEY_BYTES to the output of int_to_bytes.py. Using test public key.")
  set(UID_PUBKEY_BYTES
    "130,99,238,192,232,47,187,99,222,116,140,101,233,231,57,188,204,204,187,241,173,147,88,60,217,7,80,217"
    CACHE STRING
    "the public key used for encryption"
    )
endif()

add_definitions(-DUID_PUBKEY_BYTES=${UID_PUBKEY_BYTES})

set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static")

add_executable(uid
  uid.cpp
  ${machine_info_src}
  )

# Disable console window popping up when running the exe from Python
# Note: this also disables the output running the exe from cmd
if (WIN32)
  set_target_properties(uid PROPERTIES LINK_FLAGS -mwindows)
endif()

target_link_libraries(uid
  ${CRYPTOPP_LIBRARIES}
  ${JSONCPP_LIBRARIES}
  ${machine_info_libs}
  )

# Strip exe to reduce file size
add_custom_command(TARGET uid
  POST_BUILD
  COMMAND ${CMAKE_STRIP} -s "$<TARGET_FILE:uid>"
  COMMENT "Stripping executable $<TARGET_FILE_NAME:uid>"
  )

if (WIN32)
  option(UID_SKIP_LEGACY "Do not build the legacy test" OFF)
  if(NOT UID_SKIP_LEGACY)
    add_subdirectory(legacy_uid)
  endif()
endif()
