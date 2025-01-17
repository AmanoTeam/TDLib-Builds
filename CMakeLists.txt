cmake_minimum_required(VERSION 3.13)

project(
	tdlib
	VERSION 0.1
	DESCRIPTION "..."
	HOMEPAGE_URL "..."
	LANGUAGES C CXX
)

set(BUILD_SHARED_LIBS ON)

set(TDLIB_SOURCE_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/submodules/tdlib")

set(LIBRESSL_SOURCE_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/submodules/libressl")
set(LIBRESSL_AUTOGEN "${LIBRESSL_SOURCE_DIRECTORY}/.autogen")

set(
	TDLIB_TARGETS
	"tdclient"
	"tdc"
	"tdjson"
)

set(
	TDLIB_DEPENDENCIES
	"zlib"
	"crypto"
	"ssl"
	# "tls"
)

set(
	TDLIB_HEADERS
	"${TDLIB_SOURCE_DIRECTORY}/td/telegram/td_json_client.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/telegram/td_log.h"
	"${CMAKE_CURRENT_BINARY_DIR}/submodules/tdlib/td/telegram/tdjson_export.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/telegram/Client.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/telegram/Log.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/generate/auto/td/telegram/td_api.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/generate/auto/td/telegram/td_api.hpp"
	"${TDLIB_SOURCE_DIRECTORY}/td/telegram/td_c_client.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/generate/auto/td/telegram/td_tdc_api_inner.h"
	"${TDLIB_SOURCE_DIRECTORY}/td/generate/auto/td/telegram/td_tdc_api.h"
)

set(
	TDLIB_HEADERS_2
	"${TDLIB_SOURCE_DIRECTORY}/td/tl/TlObject.h"
)

set(
	ALL_TARGETS
	${TDLIB_TARGETS}
	${TDLIB_DEPENDENCIES}
)

set(CMAKE_POLICY_DEFAULT_CMP0069 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0048 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)

set(CMAKE_PLATFORM_NO_VERSIONED_SONAME ON)

file(READ "${CMAKE_SOURCE_DIR}/submodules/zlib/CMakeLists.txt" FILE_CONTENTS)
string(REPLACE "(UNIX)" "(1)" FILE_CONTENTS "${FILE_CONTENTS}")
string(REPLACE "(NOT APPLE)" "(0)" FILE_CONTENTS "${FILE_CONTENTS}")
file(WRITE "${CMAKE_SOURCE_DIR}/submodules/zlib/CMakeLists.txt" "${FILE_CONTENTS}")

file(READ "${CMAKE_SOURCE_DIR}/submodules/tdlib/tdactor/CMakeLists.txt" FILE_CONTENTS)
string(REPLACE "(example " "(f76a5d4 " FILE_CONTENTS "${FILE_CONTENTS}")
file(WRITE "${CMAKE_SOURCE_DIR}/submodules/tdlib/tdactor/CMakeLists.txt" "${FILE_CONTENTS}")

file(READ "${CMAKE_SOURCE_DIR}/submodules/tdlib/CMakeLists.txt" FILE_CONTENTS)
string(REPLACE "DEPENDS tl_generate_mtproto" "DEPENDS tl_generate_c tl_generate_mtproto" FILE_CONTENTS "${FILE_CONTENTS}")
string(REPLACE "tdc STATIC EXCLUDE_FROM_ALL " "tdc SHARED " FILE_CONTENTS "${FILE_CONTENTS}")
string(REPLACE "tdclient STATIC " "tdclient SHARED " FILE_CONTENTS "${FILE_CONTENTS}")
file(WRITE "${CMAKE_SOURCE_DIR}/submodules/tdlib/CMakeLists.txt" "${FILE_CONTENTS}")

set(OPENSSL_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/include")

set(OPENSSL_CRYPTO_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}crypto${CMAKE_SHARED_LIBRARY_SUFFIX}")
set(OPENSSL_SSL_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}ssl${CMAKE_SHARED_LIBRARY_SUFFIX}")

file(WRITE "${OPENSSL_SSL_LIBRARY}" "")
file(WRITE "${OPENSSL_CRYPTO_LIBRARY}" "")

set(
	ZLIB_INCLUDE_DIR
	"${CMAKE_CURRENT_SOURCE_DIR}/submodules/zlib"
	"${CMAKE_CURRENT_BINARY_DIR}/submodules/zlib"
	CACHE INTERNAL "" FORCE
)

set(ZLIB_LIBRARY "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}z${CMAKE_SHARED_LIBRARY_SUFFIX}")

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM ONLY)

if (ANDROID)
	set(ANDROID_STL "c++_static")
endif()

if (NOT EXISTS "${LIBRESSL_AUTOGEN}")
	message("-- Generating LibreSSL files")
	
	execute_process(
		COMMAND "${LIBRESSL_SOURCE_DIRECTORY}/autogen.sh"
		COMMAND_ERROR_IS_FATAL ANY
		WORKING_DIRECTORY "${LIBRESSL_SOURCE_DIRECTORY}"
	)
	
	file(WRITE "${LIBRESSL_AUTOGEN}" "")
endif()

add_subdirectory(submodules/zlib EXCLUDE_FROM_ALL)
add_subdirectory(submodules/libressl EXCLUDE_FROM_ALL)
add_subdirectory(submodules/tdlib EXCLUDE_FROM_ALL)

file(REMOVE "${OPENSSL_SSL_LIBRARY}")
file(REMOVE "${OPENSSL_CRYPTO_LIBRARY}")

foreach(dependency ${TDLIB_DEPENDENCIES})
	add_custom_command(
		OUTPUT "${dependency}"
		COMMAND "${CMAKE_COMMAND}" --build ./ --target "${dependency}"
	)
	
	add_custom_target(
		"ensure_${dependency}" ALL DEPENDS "${dependency}"
	)
endforeach()

foreach(target ${TDLIB_TARGETS})
	foreach(dependency ${TDLIB_DEPENDENCIES})
		add_dependencies(
			${target}
			"ensure_${dependency}"
		)
	endforeach()
	
	if (NOT ANDROID)
		target_link_options(
			${target}
			BEFORE
			PRIVATE
			-static-libstdc++
			-static-libgcc
		)
	endif()
endforeach()

foreach(property RUNTIME_OUTPUT_DIRECTORY LIBRARY_OUTPUT_DIRECTORY)
	foreach(target ${ALL_TARGETS})
		set_target_properties(
			${target}
			PROPERTIES
			${property} "${CMAKE_CURRENT_BINARY_DIR}"
		)
	endforeach()
endforeach()

foreach(target tdclient tdjson tdc ssl)
	foreach(property BUILD_RPATH INSTALL_RPATH)
		set_target_properties(
			${target}
			PROPERTIES
			${property} "$ORIGIN"
		)
	endforeach()
endforeach()

foreach(target ${ALL_TARGETS})
	install(
		TARGETS ${target}
		RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
	)
endforeach()

install(
	FILES ${TDLIB_HEADERS}
	DESTINATION "include/td/telegram"
)

install(
	FILES ${TDLIB_HEADERS_2}
	DESTINATION "include/td/tl"
)
