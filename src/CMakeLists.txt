if (UNIX)
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -L${PG_LIBDIR}")
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -L${PG_LIBDIR}")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${PG_CFLAGS} -fvisibility=hidden")
  set(CMAKE_CPP_FLAGS "${CMAKE_CPP_FLAGS} ${PG_CPPFLAGS}")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g")
endif (UNIX)

if (APPLE)
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -multiply_defined suppress")
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -multiply_defined suppress -Wl,-undefined,dynamic_lookup -Wl,-dead_strip_dylibs -bundle_loader ${PG_BINDIR}/postgres")
elseif (WIN32)
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /MANIFEST:NO")
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} /MANIFEST:NO")
endif (APPLE)

# Only Windows and FreeBSD need the base include/ dir instead of include/server/, and including
# both causes problems on Ubuntu where they frequently get out of sync
if (WIN32 OR (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD"))
    include_directories (${CMAKE_CURRENT_BINARY_DIR} ${PG_INCLUDEDIR} ${PG_INCLUDEDIR_SERVER})
else ()
    include_directories (${CMAKE_CURRENT_BINARY_DIR} ${PG_INCLUDEDIR_SERVER})
endif ()

if (WIN32)
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${PG_LIBDIR}/postgres.lib ws2_32.lib Version.lib")
  set(CMAKE_C_FLAGS "-D_CRT_SECURE_NO_WARNINGS")
  include_directories(${PG_INCLUDEDIR_SERVER}/port/win32)

if (MSVC)
  include_directories(${PG_INCLUDEDIR_SERVER}/port/win32_msvc)
endif (MSVC)
endif (WIN32)

set(HEADERS
  cache.h
  catalog.h
  chunk.h
  chunk_adaptive.h
  chunk_constraint.h
  chunk_dispatch.h
  chunk_dispatch_info.h
  chunk_dispatch_plan.h
  chunk_dispatch_state.h
  chunk_index.h
  chunk_insert_state.h
  compat.h
  compat-endian.h
  compat-msvc-enter.h
  compat-msvc-exit.h
  constraint_aware_append.h
  copy.h
  dimension.h
  dimension_slice.h
  dimension_vector.h
  errors.h
  event_trigger.h
  extension.h
  export.h
  extension_constants.h
  guc.h
  hypercube.h
  hypertable_cache.h
  hypertable.h
  hypertable_insert.h
  hypertable_restrict_info.h
  indexing.h
  installation_metadata.h
  partitioning.h
  planner_utils.h
  planner_import.h
  plan_expand_hypertable.h
  plan_add_hashagg.h
  process_utility.h
  scanner.h
  subspace_store.h
  tablespace.h
  trigger.h
  utils.h
  version.h)

set(SOURCES
  agg_bookend.c
  cache.c
  cache_invalidate.c
  catalog.c
  chunk.c
  chunk_adaptive.c
  chunk_constraint.c
  chunk_dispatch.c
  chunk_dispatch_info.c
  chunk_dispatch_plan.c
  chunk_dispatch_state.c
  chunk_index.c
  chunk_insert_state.c
  constraint_aware_append.c
  copy.c
  dimension.c
  dimension_slice.c
  dimension_vector.c
  event_trigger.c
  extension.c
  guc.c
  histogram.c
  hypercube.c
  hypertable.c
  hypertable_cache.c
  hypertable_insert.c
  hypertable_restrict_info.c
  indexing.c
  init.c
  installation_metadata.c
  partitioning.c
  planner.c
  plan_expand_hypertable.c
  plan_add_hashagg.c
  planner_import.c
  planner_utils.c
  process_utility.c
  scanner.c
  sort_transform.c
  subspace_store.c
  tablespace.c
  trigger.c
  utils.c
  version.c)

set(GITCOMMIT_H ${CMAKE_CURRENT_BINARY_DIR}/gitcommit.h)

# Add test source code in Debug builds
if (CMAKE_BUILD_TYPE MATCHES Debug)
  set(TEST_SOURCES ../test/src/symbol_conflict.c)
  set(TS_DEBUG 1)
  set(DEBUG 1)
endif (CMAKE_BUILD_TYPE MATCHES Debug)

if (WIN32)
  add_custom_command(
    OUTPUT ${GITCOMMIT_H}
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMAND ${CMAKE_COMMAND} -E echo_append "#define EXT_GIT_COMMIT " > ${GITCOMMIT_H}
    COMMAND (${GIT_EXECUTABLE} describe --abbrev=4 --dirty --always --tags 2> $null || call && echo "${PROJECT_VERSION_MOD}") >> ${GITCOMMIT_H}
    COMMENT "Generating gitcommit.h"
    VERBATIM)
else ()
  add_custom_command(
    OUTPUT ${GITCOMMIT_H}
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMAND ${CMAKE_COMMAND} -E echo_append "#define EXT_GIT_COMMIT " > ${GITCOMMIT_H}
    COMMAND sh -c "if `${GIT_EXECUTABLE} status > /dev/null 2>&1`; then ${GIT_EXECUTABLE} describe --abbrev=4 --dirty --always --tags ; else echo ${PROJECT_VERSION_MOD} ; fi" >> ${GITCOMMIT_H}
    COMMENT "Generating gitcommit.h"
    VERBATIM)
endif (WIN32)

add_library(${PROJECT_NAME} MODULE ${SOURCES} ${TEST_SOURCES} ${GITCOMMIT_H})

if(CMAKE_BUILD_TYPE MATCHES Debug)
  add_subdirectory(../test/src "${CMAKE_CURRENT_BINARY_DIR}/test/src")
endif(CMAKE_BUILD_TYPE MATCHES Debug)

set_target_properties(${PROJECT_NAME} PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME}-${PROJECT_VERSION_MOD}
    PREFIX "")

if (CMAKE_BUILD_TYPE MATCHES Debug)
  # This define generates extension-specific code for symbol conflict testing
  target_compile_definitions(${PROJECT_NAME} PRIVATE MODULE_NAME=${PROJECT_NAME})
endif (CMAKE_BUILD_TYPE MATCHES Debug)

install(
  TARGETS ${PROJECT_NAME}
  DESTINATION ${PG_PKGLIBDIR})

if (USE_OPENSSL)
  set(TS_USE_OPENSSL ${USE_OPENSSL})
  target_include_directories(${PROJECT_NAME} PUBLIC ${OPENSSL_INCLUDE_DIR})
  target_link_libraries(${PROJECT_NAME} ${OPENSSL_LIBRARIES})
endif (USE_OPENSSL)

configure_file(config.h.in config.h)

include_directories(${CMAKE_CURRENT_SOURCE_DIR})
add_subdirectory(bgw)
add_subdirectory(net)
add_subdirectory(telemetry)
add_subdirectory(loader)
