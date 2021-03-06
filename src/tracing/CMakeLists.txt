# we are in "src/tracing", so create the output dir manually.
# the source files include the tracing headers like
# #include "tracing/oprequest.h". so better put them into
# ${PROJECT_BINARY_DIR}/include, where acconfig.h is also located
set(header_dir ${CMAKE_BINARY_DIR}/include/tracing)
file(MAKE_DIRECTORY ${header_dir})

file(GLOB tps "*.tp")
foreach(tp ${tps})
  get_filename_component(name ${tp} NAME_WE)
  set(header ${header_dir}/${name}.h)
  set(src ${header_dir}/${name}.c)
  add_custom_command(
    OUTPUT ${header} ${src}
    COMMAND ${LTTNG_GEN_TP} ${tp} -o tracing/${name}.h -o tracing/${name}.c
    DEPENDS ${tp}
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/include
    COMMENT "generating ${header} ${src}")
  set_source_files_properties(${header} ${src}
    PROPERTIES GENERATED TRUE)
  add_custom_target(
    ${name}-tp
    DEPENDS ${header})
endforeach()

function(add_tracing_library name tracings version)
  foreach(tp_file ${tracings})
    get_filename_component(tp ${tp_file} NAME_WE)
    list(APPEND srcs
      ${header_dir}/${tp}.h
      ${header_dir}/${tp}.c)
  endforeach()
  add_library(${name} SHARED ${srcs})
  target_link_libraries(${name} ${LTTNG_LIBRARIES} dl)
  string(REGEX MATCH "^[0-9]+" soversion ${version})
  string(REGEX REPLACE "^lib" "" output_name ${name})
  set_target_properties(${name} PROPERTIES
    OUTPUT_NAME ${output_name}
    VERSION ${version}
    SOVERSION ${soversion})
endfunction()

set(osd_traces oprequest.tp osd.tp pg.tp)
add_tracing_library(libosd_tp "${osd_traces}" 1.0.0)
add_tracing_library(librados_tp librados.tp 2.0.0)
add_tracing_library(librbd_tp librbd.tp 1.0.0)
add_tracing_library(libos_tp objectstore.tp 1.0.0)

install(TARGETS librados_tp libosd_tp librbd_tp libos_tp DESTINATION lib)

