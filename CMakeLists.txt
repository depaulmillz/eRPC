cmake_minimum_required(VERSION 3.0)

project(eRPC)

find_package(GTest CONFIG REQUIRED)
find_package(GFlags CONFIG REQUIRED)

include(CMakeDependentOption)

# Pick a compiler
#set(CMAKE_CXX_COMPILER clang++)
#set(CMAKE_CXX_COMPILER g++)

add_definitions(-std=c++11 -march=native -g)
add_definitions(-Wall -Wextra -Werror -pedantic -Wsign-conversion -Wold-style-cast)
add_definitions(-Wno-error=format-zero-length -Wno-unused-function -Wno-deprecated-declarations)

set(LIBRARIES ${LIBRARIES} rt numa pthread)

# Unit tests
enable_testing()

include_directories(${CMAKE_SOURCE_DIR}/src)
include_directories(SYSTEM ${CMAKE_SOURCE_DIR}/third_party)

# DPDK may be needed for different reasons (e.g., the transport is DPDK, or
# if the transport is not DPDK but the application needs DPDK libraries)
set(DPDK_NEEDED "false")

# Options exposed to the user
set(TRANSPORT "dpdk" CACHE STRING "Datapath transport (infiniband/raw/dpdk)")
option(ROCE "Use RoCE if TRANSPORT is infiniband" OFF)
option(AZURE "Configure DPDK for Azure if TRANSPORT is dpdk" OFF)
option(PERF "Compile for performance" ON)
set(PGO "none" CACHE STRING "Profile-guided optimization (generate/use/none)")
set(LOG_LEVEL "warn" CACHE STRING "Logging level (none/error/warn/info/reorder/trace/cc)") 

# Parse the user-exposed options
if(PERF)
  message(STATUS "Compilation optimized for performance.")
  set(DEBUG OFF)
  set(TESTING OFF)
else(PERF)
  message(STATUS "Compilation not optimized for performance.")
  set(DEBUG ON)
  set(TESTING ON)
endif(PERF)

# Profile-guided optimization
if(PGO STREQUAL "generate")
  message(STATUS "Profile-guided optimization (generate mode) is enabled. Performance will be low.")
  add_definitions(-fprofile-generate)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-generate")
elseif(PGO STREQUAL "use")
  message(STATUS "Profile-guided optimization (use mode) is enabled.")
  add_definitions(-fprofile-use -fprofile-correction)
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-use -fprofile-correction")
elseif(PGO STREQUAL "none")
  message(STATUS "Profile-guided optimization is disabled.")
endif()

# Logging level
if(LOG_LEVEL STREQUAL "none")
  message(STATUS "Logging level = none.")
  add_definitions(-DERPC_LOG_LEVEL=0)
elseif(LOG_LEVEL STREQUAL "error")
  message(STATUS "Logging level = error.")
  add_definitions(-DERPC_LOG_LEVEL=1)
elseif(LOG_LEVEL STREQUAL "warn")
  message(STATUS "Logging level = warn.")
  add_definitions(-DERPC_LOG_LEVEL=2)
elseif(LOG_LEVEL STREQUAL "info")
  message(STATUS "Logging level = info.")
  add_definitions(-DERPC_LOG_LEVEL=3)
elseif(LOG_LEVEL STREQUAL "reorder")
  message(STATUS "Logging level = reorder. Warning: Performance will be low.")
  add_definitions(-DERPC_LOG_LEVEL=4)
elseif(LOG_LEVEL STREQUAL "trace")
  message(STATUS "Logging level = trace. Warning: Performance will be low.")
  add_definitions(-DERPC_LOG_LEVEL=5)
elseif(LOG_LEVEL STREQUAL "cc")
  message(STATUS "Logging level = cc. Warning: Performance will be low.")
  add_definitions(-DERPC_LOG_LEVEL=6)
else()
  message(STATUS "No logging level specified. Using warning level.")
  add_definitions(-DERPC_LOG_LEVEL=2)
endif()

# Debug mode
if(DEBUG)
  message(STATUS "Debugging is enabled. Perf will be low.")
else(DEBUG)
  message(STATUS "Debugging is disabled.")
  add_definitions(-DNDEBUG)
  add_definitions(-O2)
endif(DEBUG)

# Testing for packet loss, machine failure, etc
if(TESTING)
  message(STATUS "Testing is enabled. Performance will be low.")
  add_definitions(-DERPC_TESTING=true)
else(TESTING)
  message(STATUS "Testing is disabled, so tests may fail.")
  add_definitions(-DERPC_TESTING=false)
endif(TESTING)

set(SOURCES
  src/nexus_impl/nexus.cc
  src/nexus_impl/nexus_bg_thread.cc
  src/nexus_impl/nexus_sm_thread.cc
  src/rpc_impl/rpc.cc
  src/rpc_impl/rpc_queues.cc
  src/rpc_impl/rpc_rfr.cc
  src/rpc_impl/rpc_cr.cc
  src/rpc_impl/rpc_kick.cc
  src/rpc_impl/rpc_req.cc
  src/rpc_impl/rpc_resp.cc
  src/rpc_impl/rpc_ev_loop.cc
  src/rpc_impl/rpc_fault_inject.cc
  src/rpc_impl/rpc_pkt_loss.cc
  src/rpc_impl/rpc_rx.cc
  src/rpc_impl/rpc_connect_handlers.cc
  src/rpc_impl/rpc_disconnect_handlers.cc
  src/rpc_impl/rpc_reset_handlers.cc
  src/rpc_impl/rpc_sm_api.cc
  src/rpc_impl/rpc_sm_helpers.cc
  src/transport_impl/transport.cc
  src/transport_impl/dpdk/dpdk_transport.cc
  src/transport_impl/dpdk/dpdk_transport_datapath.cc
  src/transport_impl/infiniband/ib_transport.cc
  src/transport_impl/infiniband/ib_transport_datapath.cc
  src/transport_impl/raw/raw_transport.cc
  src/transport_impl/raw/raw_transport_datapath.cc
  src/util/huge_alloc.cc
  src/util/externs.cc
  src/util/tls_registry.cc)

# Transport-specific. Mellanox OFED drivers are the best choice for raw and
# infiniband, but they do not play well with DPDK. So we compile only one
# transport. Other transports are exluded using preprocessor macros.
string(TOUPPER ${TRANSPORT} DEFINE_TRANSPORT)
add_definitions(-DERPC_${DEFINE_TRANSPORT}=true)
message(STATUS "Selected transport = ${TRANSPORT}.")
set(CONFIG_IS_ROCE false)

if("${TRANSPORT}" STREQUAL "dpdk")
  set(CONFIG_TRANSPORT "DpdkTransport")
  set(CONFIG_HEADROOM 40)
  set(DPDK_NEEDED "true") # We'll resolve DPDK later
else()
  find_library(IBVERBS_LIB ibverbs)
  if(NOT IBVERBS_LIB)
    message(FATAL_ERROR "ibverbs library not found")
  endif()

  set(LIBRARIES ${LIBRARIES} ibverbs)
  if(TRANSPORT STREQUAL "raw")
    set(CONFIG_TRANSPORT "RawTransport")
    set(CONFIG_HEADROOM 40)
  elseif(TRANSPORT STREQUAL "infiniband")
    set(CONFIG_TRANSPORT "IBTransport")
    if(ROCE)
      set(CONFIG_HEADROOM 40)
      set(CONFIG_IS_ROCE true)
    else()
      set(CONFIG_HEADROOM 0)
      set(CONFIG_IS_ROCE false)
    endif()
  endif()
endif()

if(AZURE)
  set(CONFIG_IS_AZURE true)
  message(STATUS "Configuring DPDK for Azure")
else()
  set(CONFIG_IS_AZURE false)
  message(STATUS "Configuring DPDK for bare-metal cluster (i.e., not Azure)")
endif()

configure_file(src/config.h.in src/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/src)

# MICA sources
set(MICA_SOURCES
  mica/src/mica/util/cityhash/city_mod.cc
  mica/src/mica/util/config.cc)

# The tests to run using ctest
set(CLIENT_TESTS
  create_session_test
  destroy_session_test
  small_msg_test
  large_msg_test
  req_in_cont_func_test
  req_in_req_func_test
  packet_loss_test
  #server_failure_test
  multi_process_test)

set(PROTOCOL_TESTS
  rpc_sm_test
  rpc_list_test
  rpc_req_test
  rpc_resp_test
  rpc_cr_test
  rpc_rfr_test
  rpc_kick_test)

if(TRANSPORT STREQUAL "raw")
  set(TRANSPORT_TESTS
    raw_transport_test)
endif()

# These are not run using ctest
set(UTIL_TESTS
  huge_alloc_test
  hugepage_caching_virt2phy_test
  timing_wheel_test
  heartbeat_mgr_test
  rand_test
  misc_test
  fixed_vector_test
  timely_test
  numautil_test)

# Compile the library
add_library(erpc ${SOURCES})
target_include_directories(erpc PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/src)
include_directories(erpc PUBLIC ${CMAKE_SOURCE_DIR}/src)
include_directories(erpc PUBLIC ${CMAKE_SOURCE_DIR}/third_party)

if(DPDK_NEEDED STREQUAL "true")
  message(STATUS "DPDK needed to build eRPC")
  find_library(DPDK CONFIG)
  if(NOT ${DPDK_FOUND})
    message(STATUS "DPDK library not found")
    set(LIBRARIES ${LIBRARIES} -Wl,--whole-archive -ldpdk -Wl,--no-whole-archive numa dl ibverbs mlx4 mlx5)
  else()
    message(STATUS "DPDK WAS FOUND with ${DPDK_INCLUDE_DIR} , ${DPDK_LIBRARIES}")

    if(NOT DEFINED DPDK_LIBRARIES)
      set(DPDK_LIBRARIES -lrte_telemetry -lrte_bpf -lrte_flow_classify -lrte_pipeline -lrte_table -lrte_port -lrte_fib -lrte_ipsec -lrte_vhost -lrte_stack -lrte_security -lrte_sched -lrte_reorder -lrte_rib -lrte_rcu -lrte_rawdev -lrte_pdump -lrte_power -lrte_member -lrte_lpm -lrte_latencystats -lrte_kni -lrte_jobstats -lrte_ip_frag -lrte_gso -lrte_gro -lrte_eventdev -lrte_efd -lrte_distributor -lrte_cryptodev -lrte_compressdev -lrte_cfgfile -lrte_bitratestats -lrte_bbdev -lrte_acl -lrte_timer -lrte_hash -lrte_metrics -lrte_cmdline -lrte_pci -lrte_ethdev -lrte_meter -lrte_net -lrte_mbuf -lrte_mempool -lrte_ring -lrte_eal -lrte_kvargs -lbsd)
      set(LIBRARIES ${LIBRARIES} ${DPDK_LIBRARIES})
    else()
      set(LIBRARIES ${LIBRARIES} -Wl,--whole-archive dpdk -Wl,--no-whole-archive numa dl ibverbs mlx4 mlx5)
    endif()

  endif()

  # DPDK include directory. Locating rte_config.h does not work on some systems.
  # Example: it may be kept in /usr/include/x86_64-linux-gnu/, and symlinked
  # from the real DPDK include directory (/usr/include/dpdk/).
  find_path(DPDK_INCLUDE_DIR NAMES rte_ethdev.h PATH_SUFFIXES dpdk)
  if (DPDK_INCLUDE_DIR)
    message(STATUS "DPDK include directory = ${DPDK_INCLUDE_DIR}")
  else()
    message(FATAL_ERROR "DPDK include directory not found")
  endif()

  find_path(DPDK_SECOND_INCLUDE_DIR NAMES rte_config.h PATH_SUFFIXES dpdk)

  include_directories(SYSTEM ${DPDK_INCLUDE_DIR} ${DPDK_SECOND_INCLUDE_DIR})
else()
  message(STATUS "DPDK not needed to build eRPC")
endif()

# Compile the tests
if(TESTING)
  foreach(test_name IN LISTS CLIENT_TESTS)
    add_executable(${test_name} tests/client_tests/${test_name}.cc)
    target_link_libraries(${test_name} PRIVATE erpc ${LIBRARIES})
    target_link_libraries(${test_name} PRIVATE GTest::gmock GTest::gtest GTest::gmock_main GTest::gtest_main)
    add_test(NAME ${test_name} COMMAND ${test_name})
  endforeach()

  foreach(test_name IN LISTS PROTOCOL_TESTS)
    add_executable(${test_name} tests/protocol_tests/${test_name}.cc)
    target_link_libraries(${test_name} PRIVATE erpc ${LIBRARIES})
    target_link_libraries(${test_name} PRIVATE GTest::gmock GTest::gtest GTest::gmock_main GTest::gtest_main)
    add_test(NAME ${test_name} COMMAND ${test_name})
  endforeach()

  foreach(test_name IN LISTS TRANSPORT_TESTS)
    add_executable(${test_name} tests/transport_tests/${test_name}.cc)
    target_link_libraries(${test_name} PRIVATE erpc ${LIBRARIES})
    target_link_libraries(${test_name} PRIVATE GTest::gmock GTest::gtest GTest::gmock_main GTest::gtest_main)
    add_test(NAME ${test_name} COMMAND ${test_name})
  endforeach()

  foreach(test_name IN LISTS UTIL_TESTS)
    add_executable(${test_name} tests/util_tests/${test_name}.cc)
    target_link_libraries(${test_name} PRIVATE erpc ${LIBRARIES})
    target_link_libraries(${test_name} PRIVATE GTest::gmock GTest::gtest GTest::gmock_main GTest::gtest_main)
  endforeach()
else()
  message(STATUS "Not testing enable with -DPERF=OFF")
endif()

# The app to compile. Only one app is compiled to reduce compile time.
if(EXISTS "${CMAKE_SOURCE_DIR}/scripts/autorun_app_file")
  file(STRINGS "scripts/autorun_app_file" APP)
else()
  message(STATUS "No autorun_app_file found. No application will be compiled.")
  return()
endif()
message(STATUS "Compiling app = " ${APP})

# Add app-specific defines now, isolating them from the library and tests

# libpmem is installable from package managers in only recent distros. If it's
# not present, don't link it in.
find_library(PMEM_LIB pmem)
if(NOT PMEM_LIB)
  message(STATUS "Persistent memory library (libpmem) not found")
  set(PMEM "")
else()
  set(PMEM "pmem")
endif()

if(APP STREQUAL "smr")
  # Raft library from willemt/raft, installed at system-level
  set(LIBRARIES ${LIBRARIES} raft ${PMEM})
  # MICA
  include_directories(mica/src)
  set(APP_ADDITIONAL_SOURCES ${MICA_SOURCES})
elseif(APP STREQUAL "masstree_analytics")
  # CMake-based Masstree library from anujkaliaiitd/masstree-beta
  link_directories(${CMAKE_SOURCE_DIR}/third_party/masstree-beta)
  include_directories(SYSTEM ${CMAKE_SOURCE_DIR}/third_party/masstree-beta)
  add_definitions(-include ${CMAKE_SOURCE_DIR}/third_party/masstree-beta/config.h)
  set(APP_ADDITIONAL_SOURCES
    apps/masstree_analytics/mt_index_api.cc
    mica/src/mica/util/cityhash/city_mod.cc)
  set(LIBRARIES ${LIBRARIES} masstree)
  include_directories(mica/src)
elseif(APP STREQUAL "mica_test")
  include_directories(mica/src)
  set(APP_ADDITIONAL_SOURCES ${MICA_SOURCES})
elseif(APP STREQUAL "latency")
  set(LIBRARIES ${LIBRARIES} ${PMEM})
elseif(APP STREQUAL "pmem_bw")
  set(LIBRARIES ${LIBRARIES} ${PMEM})
  message(STATUS "DPDK is needed for pmem_bw application")
  add_definitions(-fpermissive)
  set(DPDK_NEEDED "true")
elseif(APP STREQUAL "persistent_kv")
  set(LIBRARIES ${LIBRARIES} ${PMEM} cityhash)
elseif(APP STREQUAL "log")
  set(LIBRARIES ${LIBRARIES} ${PMEM})
endif()


# Using link-time optimization sometimes requires building with sources instead
# of liberpc. See the hello world example's Makefile for an example of
# compiling with liberpc.
add_executable(${APP} apps/${APP}/${APP}.cc ${APP_ADDITIONAL_SOURCES} ${SOURCES})
target_compile_options(${APP} PRIVATE "-flto")
target_link_libraries(${APP} ${LIBRARIES} ${GTEST_LIBRARIES} gflags_static)

install(TARGETS erpc
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin
        INCLUDES DESTINATION include
        )

set(CPACK_PACKAGE_VENDOR "dePaul Miller")
SET(CPACK_GENERATOR "DEB")
SET(CPACK_DEBIAN_PACKAGE_MAINTAINER "dePaul Miller")
set(CPACK_SOURCE_GENERATOR "TGZ;ZIP")
set(CPACK_SOURCE_IGNORE_FILES
        /.git
        /dist
        /.*build.*
        /\\\\.DS_Store
        )
include(CPack)