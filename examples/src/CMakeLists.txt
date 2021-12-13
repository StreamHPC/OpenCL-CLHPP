set(OPENCL_HEADERS_INCLUDE_DIR_GENEXPR $<TARGET_PROPERTY:OpenCL::Headers,INTERFACE_INCLUDE_DIRECTORIES>)
###############################
###                         ###
### Workaround to CMake bug ###
###                         ###
###############################
#
# The generated mocks\Mockcl.h include the OpenCL headers as
#
# #include "CL/cl_platform.h"
# #include "cl.h"
#
# which requires the tests to not only inherit the INTERFACE_INCLUDE_DIRECTORIES, but also the same dir with
# /CL appended at the end. There seems to be a bug in CMake (3.19 even) where if a target has the same genexpr
# predicate twice, the second time it always fails. Having both
#
# $<TARGET_PROPERTY:Headers,INTERFACE_INCLUDE_DIRECTORIES> (invisbly via linking to OpenCL::Headers)
# $<TARGET_PROPERTY:Headers,INTERFACE_INCLUDE_DIRECTORIES>/CL
#
# means we can only get one of these switches. We forcibly go around it by extracting the value manually.
#
# Bug ticket: https://gitlab.kitware.com/cmake/cmake/-/issues/22735
#
# ticket closed with by-design. We should find a way to fix the Mock tests to not consume the public API using
# mixed style includes.
get_target_property(OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES OpenCL::Headers INTERFACE_INCLUDE_DIRECTORIES)

if(OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES MATCHES "BUILD_INTERFACE")
  # When building OpenCL::Headers is part of this build (OpenCL-SDK), we have a property like for eg.
  # $<BUILD_INTERFACE:C:/OpenCL-SDK/external/OpenCL-Headers>;$<INSTALL_INTERFACE:include>
  # and need to touch up this property. We want the BUILD_INTERFACE part only.
  # When OpenCL::Headers is consumed through a Package Config file (OpenCL-CLHPP), this property
  # no longer holds unevaluated generator expressions.
  string(REGEX MATCHALL
    [[\$<BUILD_INTERFACE:(.*)>;]]
    OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES
    "${OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES}")
  set(OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_MATCH_1}")
endif()
# End of workaround

foreach(EXAMPLE IN ITEMS
  trivial
  trivialSizeTCompat
  headerexample
)
  add_executable(${EXAMPLE} ${EXAMPLE}.cpp)
  target_link_libraries(${EXAMPLE}
    PRIVATE
      OpenCL::Headers
      OpenCL::OpenCL
      Threads::Threads
  )
  target_include_directories(${EXAMPLE}
    PRIVATE
      ${PROJECT_SOURCE_DIR}/include

      ${OPENCL_HEADERS_INTERFACE_INCLUDE_DIRECTORIES}/CL
      #${OPENCL_HEADERS_INCLUDE_DIR_GENEXPR}/CL
  )
  target_compile_definitions(${EXAMPLE}
    PRIVATE
      CL_HPP_TARGET_OPENCL_VERSION=300
      CL_HPP_ENABLE_EXCEPTIONS
  )
endforeach()
