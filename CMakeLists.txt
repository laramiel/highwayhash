
project(highwayhash C CXX)

cmake_minimum_required(VERSION 3.18)


if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
  set(ARCH "x86_64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "amd64")
  set(ARCH "x86_64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64")
  # cmake reports AMD64 on Windows, but we might be building for 32-bit.
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH "x86_64")
  else()
    set(ARCH "x86")
  endif()
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
  set(ARCH "x86")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "i386")
  set(ARCH "x86")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "i686")
  set(ARCH "x86")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  set(ARCH "aarch64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
  set(ARCH "aarch64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64e")
  set(ARCH "aarch64")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm*")
  set(ARCH "arm")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "mips")
  # Just to avoid the “unknown processor” error.
  set(ARCH "generic")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
  set(ARCH "ppc64le")
else()
  message(FATAL_ERROR "Unknown processor:" ${CMAKE_SYSTEM_PROCESSOR})
endif()


if(CMAKE_COMPILER_IS_GNUCXX OR CLANG)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -O3 -fPIC -pthread -Wno-maybe-uninitialized")
  if(ARCH STREQUAL "arm")
    # aarch64 and ARM use the same code, although ARM usually needs an extra flag for NEON.
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfloat-abi=hard -march=armv7-a -mfpu=neon")
  endif()
endif()


if(ARCH STREQUAL "aarch64" OR ARCH STREQUAL "arm")
  set(HH_ARCH_SOURCES
    highwayhash/hh_neon.cc
  )
  set(HH_ARCH_SOURCES
    highwayhash/hh_neon.h
  )

elseif(ARCH STREQUAL "ppc64le")
  set(HH_ARCH_SOURCES
    highwayhash/hh_vsx.cc
  )
  set(HH_ARCH_INCLUDES 
    highwayhash/hh_vsx.h
  )

  set_source_files_properties( highwayhash/benchmark.cc PROPERTIES COMPILE_FLAGS -mvsx)
  set_source_files_properties( highwayhash/hh_vsx.cc PROPERTIES COMPILE_FLAGS -mvsx)


elseif(ARCH STREQUAL "x86" OR ARCH STREQUAL "x86_64")
  set(HH_ARCH_SOURCES
    highwayhash/hh_avx2.cc
    highwayhash/hh_sse41.cc
  )
  set(HH_ARCH_INCLUDES
    highwayhash/hh_avx2.h
    highwayhash/hh_sse41.h
  )

  # TODO: Portability: Have AVX2 be optional so benchmarking can be done on older machines.
  set_source_files_properties( highwayhash/benchmark.cc  PROPERTIES COMPILE_FLAGS  -mavx2)
  set_source_files_properties( highwayhash/sip_tree_hash.cc  PROPERTIES COMPILE_FLAGS  -mavx2)
  set_source_files_properties( highwayhash/hh_avx2.cc  PROPERTIES COMPILE_FLAGS  -mavx2)
  set_source_files_properties( highwayhash/hh_sse41.cc  PROPERTIES COMPILE_FLAGS  -msse4.1)
  set_source_files_properties( highwayhash/hh_portable.cc  PROPERTIES COMPILE_FLAGS  -DHH_TARGET_NAME=Portable)

else()
  # Unknown architecture.
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DHH_DISABLE_TARGET_SPECIFIC")
endif()


add_library(highwayhash)
target_sources(highwayhash PRIVATE

  highwayhash/c_bindings.h
  highwayhash/hh_portable.h
  highwayhash/state_helpers.h
  highwayhash/c_bindings.cc
  highwayhash/hh_portable.cc

  highwayhash/arch_specific.h
  highwayhash/compiler_specific.h
  highwayhash/arch_specific.cc
  highwayhash/load3.h  
  highwayhash/vector128.h  
  highwayhash/vector256.h  
  highwayhash/endianess.h  
  highwayhash/iaca.h  
  highwayhash/hh_types.h
  highwayhash/hh_buffer.h

  ${HH_ARCH_INCLUDES}
  ${HH_ARCH_SOURCES}

  highwayhash/scalar_sip_tree_hash.h
  highwayhash/sip_hash.h
  highwayhash/sip_tree_hash.h
  highwayhash/scalar_sip_tree_hash.cc
  highwayhash/sip_hash.cc
  highwayhash/sip_tree_hash.cc
)
target_include_directories(highwayhash PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

if(NOT WIN32 AND NOT ANDROID)
  target_link_libraries(highwayhash pthread)
endif()


# Tests

add_library(nanobenchmark)
target_sources(nanobenchmark PRIVATE
   highwayhash/nanobenchmark.h
   highwayhash/nanobenchmark.cc

   highwayhash/instruction_sets.h
   highwayhash/os_specific.h  
   highwayhash/profiler.h  
   highwayhash/tsc_timer.h  

   highwayhash/instruction_sets.cc
   highwayhash/os_specific.cc
)
target_include_directories(nanobenchmark PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})


add_executable(highwayhash_test)
target_sources(highwayhash_test PRIVATE

  highwayhash/highwayhash_test.cc
  highwayhash/highwayhash_test_portable.cc
  highwayhash/highwayhash_test_target.h
)
target_link_libraries(highwayhash_test highwayhash nanobenchmark)


add_executable(vector_test)
target_sources(vector_test PRIVATE
   highwayhash/vector_test.cc
   highwayhash/vector_test_portable.cc
   highwayhash/vector_test_target.h
)
target_link_libraries(vector_test highwayhash nanobenchmark)



if(ARCH STREQUAL "aarch64" OR ARCH STREQUAL "arm")
  target_sources(highwayhash_test PRIVATE
    highwayhash/highwayhash_test_neon.cc
  )
  target_sources(vector_test PRIVATE
    highwayhash/vector_test_neon.cc
  )

elseif(ARCH STREQUAL "x86" OR ARCH STREQUAL "x86_64")
  target_sources(highwayhash_test PRIVATE
    highwayhash/highwayhash_test_avx2.cc
    highwayhash/highwayhash_test_sse41.cc
  )
  target_sources(vector_test PRIVATE
    highwayhash/vector_test_avx2.cc
    highwayhash/vector_test_sse41.cc
  )

  set_source_files_properties( highwayhash/highwayhash_test_avx2.cc  PROPERTIES COMPILE_FLAGS  -mavx2)
  set_source_files_properties( highwayhash/highwayhash_test_sse41.cc  PROPERTIES COMPILE_FLAGS  -msse4.1)
  set_source_files_properties( highwayhash/vector_test_avx2.cc  PROPERTIES COMPILE_FLAGS  -mavx2)
  set_source_files_properties( highwayhash/vector_test_sse41.cc  PROPERTIES COMPILE_FLAGS  -msse4.1)


elseif(ARCH STREQUAL "ppc64le")
  target_sources(highwayhash_test PRIVATE
    highwayhash/highwayhash_test_vsx.cc
  )

  set_source_files_properties( highwayhash/highwayhash_test_vsx.cc PROPERTIES COMPILE_FLAGS -mvsx)
  set_source_files_properties( highwayhash/vector_test.cc PROPERTIES COMPILE_FLAGS  -DHH_DISABLE_TARGET_SPECIFIC)

endif()


add_executable(sip_hash_test)
target_sources(sip_hash_test PRIVATE

   highwayhash/sip_hash_test.cc
)
target_link_libraries(sip_hash_test highwayhash)



add_executable(example)
target_sources(example PRIVATE
   highwayhash/example.cc
 )
target_link_libraries(example highwayhash)

