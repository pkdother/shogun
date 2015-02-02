if(SYSTEM_INCLUDES)
  INCLUDE_DIRECTORIES(SYSTEM ${SYSTEM_INCLUDES})
endif()
include_directories(${INCLUDES})

FILE(GLOB_RECURSE LIBSHOGUN_SRC *.${EXT_SRC_CPP} *.${EXT_SRC_C})
FILE(GLOB_RECURSE LIBSHOGUN_HEADERS *.${EXT_SRC_HEADER})
FILE(GLOB_RECURSE LIBSHOGUN_SRC_TMP *.${EXT_CPP_TMP})

LIST(APPEND LIBSHOGUN_HEADERS "${CMAKE_CURRENT_BINARY_DIR}/lib/config.h")

IF (PROTOBUF_FOUND)
	FILE(GLOB protobuf_src ${CMAKE_CURRENT_SOURCE_DIR}/io/protobuf/*.proto)

	FOREACH(FIL ${protobuf_src})
		get_filename_component(FIL_WE ${FIL} NAME_WE)
		SET(name "${CMAKE_CURRENT_BINARY_DIR}/io/protobuf/${FIL_WE}")

		if (EXISTS "${name}.pb.cc")
			file(REMOVE "${name}.pb.cc")
		endif()
		if (EXISTS "${name}.pb.h")
			file(REMOVE "${name}.pb.h")
		endif()

		list(APPEND LIBSHOGUN_SRC "${name}.pb.cc")
		list(APPEND LIBSHOGUN_HEADERS "${name}.pb.h")
	ENDFOREACH()
ENDIF()

# add target to compile the libshogun sources
add_library(libshogun OBJECT base/class_list.cpp ${LIBSHOGUN_SRC})
set_target_properties(libshogun PROPERTIES
    COMPILE_DEFINITIONS "${DEFINES}"
    POSITION_INDEPENDENT_CODE TRUE)
IF (SANITIZER_FLAGS)
  set_target_properties(libshogun PROPERTIES COMPILE_FLAGS ${SANITIZER_FLAGS})
ENDIF()

IF (PROTOBUF_FOUND)
	file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/io/protobuf)
	FOREACH(FIL ${protobuf_src})
		get_filename_component(FIL_WE ${FIL} NAME_WE)

		SET(name "${CMAKE_CURRENT_BINARY_DIR}/io/protobuf/${FIL_WE}")

		ADD_CUSTOM_COMMAND(
			 OUTPUT "${name}.pb.cc" "${name}.pb.h"
			 COMMAND ${PROTOBUF_PROTOC_EXECUTABLE} ${FIL}
				 --cpp_out ${CMAKE_CURRENT_BINARY_DIR}/io/protobuf
				 --proto_path ${CMAKE_CURRENT_SOURCE_DIR}/io/protobuf
			 COMMENT "Running C++ protocol buffer compiler on ${FIL_WE}.proto"
			)
		ADD_CUSTOM_TARGET("${FIL_WE}Protobuf" DEPENDS "${name}.pb.cc")
		add_dependencies(libshogun "${FIL_WE}Protobuf")
	ENDFOREACH()
ENDIF()

# Generate versionstring.h
ADD_CUSTOM_TARGET(
    version
    ${CMAKE_COMMAND} -D SRC=${CMAKE_CURRENT_SOURCE_DIR}/lib/versionstring.h.in
                     -D DST=${CMAKE_CURRENT_BINARY_DIR}/lib/versionstring.h
                     -D ROOT_DIR=${CMAKE_SOURCE_DIR}
                     -D PYTHON_EXECUTABLE=${PYTHON_EXECUTABLE}
                     -P ${CMAKE_MODULE_PATH}/version.cmake
    COMMENT "Generating version header"
)
add_dependencies(libshogun version)

# generate source files from templates
FOREACH(template ${LIBSHOGUN_SRC_TMP})
  FILE(RELATIVE_PATH template ${CMAKE_CURRENT_SOURCE_DIR} "${template}")
  STRING(REGEX REPLACE ${EXT_CPP_TMP} ${EXT_SRC_CPP} generated_cpp "${template}")
  STRING(REGEX REPLACE ${EXT_CPP_TMP} ${EXT_CPP_PY} generator_script "${template}")
  STRING(REGEX REPLACE ".*/(.*).${EXT_CPP_TMP}" "\\1" generated_target "${template}")
  CONFIGURE_FILE(${template} ${generated_cpp} COPYONLY)
  CONFIGURE_FILE(${generator_script}.in ${generator_script} @ONLY)

  IF(MSVC)
    FOREACH(h ${LIBSHOGUN_HEADERS})
      FILE(APPEND ${CMAKE_CURRENT_BINARY_DIR}/headers_list.txt "${h}\n")
    ENDFOREACH()

    ADD_CUSTOM_COMMAND(OUTPUT ${generated_cpp}
        COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/${generator_script} ${CMAKE_CURRENT_BINARY_DIR}/${generated_cpp} -in ${CMAKE_CURRENT_BINARY_DIR}/headers_list.txt
        DEPENDS ${template} ${LIBSHOGUN_HEADERS} ${CMAKE_CURRENT_BINARY_DIR}/headers_list.txt
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating ${generated_cpp}"
    )
  ELSE()
    ADD_CUSTOM_COMMAND(OUTPUT ${generated_cpp}
        COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/${generator_script} ${CMAKE_CURRENT_BINARY_DIR}/${generated_cpp} ${LIBSHOGUN_HEADERS}
        DEPENDS ${template} ${LIBSHOGUN_HEADERS}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating ${generated_cpp}"
    )
  ENDIF()

  ADD_CUSTOM_TARGET(${generated_target} DEPENDS ${generated_cpp})
  add_dependencies(libshogun ${generated_target})
ENDFOREACH()

# add target for dynamic
add_library(shogun SHARED $<TARGET_OBJECTS:libshogun> lib/config.h)
set_target_properties(shogun PROPERTIES
                      VERSION ${LIBSHOGUNVER}
                      SOVERSION ${LIBSHOGUNSO})

target_link_libraries(shogun ${POSTLINKFLAGS})

# add bundled libraries to dependency
if(SHOGUN_DEPENDS)
  add_dependencies(libshogun ${SHOGUN_DEPENDS})
endif()

# add target for static library if enabled
if (BUILD_STATIC)
  add_library(shogun-static STATIC $<TARGET_OBJECTS:libshogun> lib/config.h)
  set_target_properties(shogun-static PROPERTIES OUTPUT_NAME shogun)
ENDIF()

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/lib/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/lib/config.h)

INSTALL(DIRECTORY ./
        DESTINATION include/shogun
        COMPONENT headers
        FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
        PATTERN ".settings" EXCLUDE)

IF (LIB_INSTALL_DIR)
  SET(SHOGUN_LIB_INSTALL ${LIB_INSTALL_DIR})
ELSE()
  SET(SHOGUN_LIB_INSTALL lib)
ENDIF()

INSTALL(TARGETS shogun
  ARCHIVE DESTINATION ${SHOGUN_LIB_INSTALL}
  LIBRARY DESTINATION ${SHOGUN_LIB_INSTALL}
  COMPONENT libraries)
