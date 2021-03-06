project(libsqrl)
set(PROJECT_NAME libsqrl)
cmake_minimum_required(VERSION 2.8)

# check for polluted source tree
if(EXISTS ${CMAKE_SOURCE_DIR}/CMakeCache.txt OR
   EXISTS ${CMAKE_SOURCE_DIR}/CMakeFiles)
 message(FATAL_ERROR
   "CMakeCache.txt or CMakeFiles exists in source directory!")
endif()
# make sure the user doesn't play dirty with symlinks
get_filename_component(srcdir "${CMAKE_SOURCE_DIR}" REALPATH)
get_filename_component(bindir "${CMAKE_BINARY_DIR}" REALPATH)
# disallow in-source builds
if(${srcdir} STREQUAL ${bindir})
 message(FATAL_ERROR "In-source builds are forbidden!")
endif()

# A hack...
if(CMAKE_MAJOR_VERSION EQUAL 3)
	cmake_policy(SET CMP0026 OLD)
	cmake_policy(SET CMP0045 OLD)
endif(CMAKE_MAJOR_VERSION EQUAL 3)
# End Hack.

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

include(ExternalProject)
include(${CMAKE_SOURCE_DIR}/cmake/find_sse.cmake)
#include(${CMAKE_SOURCE_DIR}/cmake/mergestaticlibs.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/libutils.cmake)

# Major version corresponds to SQRL protocol version
set(sqrl_version_major 1)
# Minor version is build year
set(sqrl_version_minor 2016)
set(sqrl_build 30)
set(sqrl_version "${sqrl_version_major}.${sqrl_version_minor} build ${sqrl_build}")

#set(CMAKE_BUILD_TYPE Release)
set(CMAKE_BUILD_TYPE Debug)

set(CMAKE_C_FLAGS_DEBUG "-DDEBUG")

enable_testing()

set(SRCDIR ${CMAKE_SOURCE_DIR}/src)

if(CMAKE_COMPILER_IS_GNUCC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fvisibility=hidden -fPIC")
endif()

CHECK_FOR_SSE()

configure_file(${SRCDIR}/config.h.in ${CMAKE_BINARY_DIR}/include/sqrl_config.h)
include_directories( ${CMAKE_BINARY_DIR}/include ${CMAKE_BINARY_DIR}/src/libsodium/src/libsodium/crypto_pwhash/scryptsalsa208sha256)

configure_file(${SRCDIR}/sqrl_depends.c.in sqrl_depends.c)
file(COPY 
	${CMAKE_SOURCE_DIR}/data/test1.sqrl
	${CMAKE_SOURCE_DIR}/data/test2.sqrl
	${CMAKE_SOURCE_DIR}/data/vectors 
	DESTINATION ${CMAKE_BINARY_DIR})
configure_file(README.in ${CMAKE_SOURCE_DIR}/README.md)


set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)

######################### MAC BUILD #########################
set(CMAKE_MACOSX_RPATH 1)
set(REQUIRED_LIBS_PRESENT TRUE)

ExternalProject_Add(libsodium
	PREFIX ${CMAKE_BINARY_DIR}
#	BINARY_DIR ${CMAKE_BINARY_DIR}/libsodium
	BUILD_IN_SOURCE 1
	GIT_REPOSITORY https://github.com/jedisct1/libsodium.git
	GIT_TAG stable
	UPDATE_COMMAND ""
	CONFIGURE_COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/scripts/sodium-conf.sh ${CMAKE_BINARY_DIR}
	BUILD_COMMAND make -s
	INSTALL_COMMAND make -s install)
add_library(sodium STATIC IMPORTED)
SET_TARGET_PROPERTIES(sodium PROPERTIES IMPORTED_LOCATION ${CMAKE_BINARY_DIR}/lib/libsodium.a)
set_target_properties(libsodium PROPERTIES FOLDER meta_build)
set(LIBS ${LIBS} sodium)

find_library(RT_LIB NAMES rt)
if(NOT ${RT_LIB} STREQUAL "RT_LIB-NOTFOUND")
	set(LIBS ${LIBS} ${RT_LIB})
endif()

find_library(PTHREAD_LIB NAMES pthread)
if(NOT ${PTHREAD_LIB} STREQUAL "PTHREAD_LIB-NOTFOUND")
	set(LIBS ${LIBS} ${PTHREAD_LIB})
endif()

include_directories(include)

set(SG_CLIENT src/client.c src/client_protocol.c src/transaction.c)
source_group(Client FILES ${SG_CLIENT})
set(SG_CLIENT_USER src/user.c src/user_storage.c src/storage.c src/block.c)
source_group(Client\\User FILES ${SG_CLIENT_USER})
set(SG_SERVER src/server.c src/server_protocol.c)
source_group(Server FILES ${SG_SERVER})
set(SG_CRYPTO src/crypto/aes.c src/crypto/gcm.c src/crypto/crypt.c src/crypto/aes.h src/crypto/gcm.h)
source_group(Crypto FILES ${SG_CRYPTO})
set(SG_UTIL src/util.c src/encdec.c src/realtime.c src/uri.c src/platform.c ${CMAKE_BINARY_DIR}/sqrl_depends.c)
source_group(Utility FILES ${SG_UTIL})
set(SG_ENTROPY src/entropy/entropy.c src/entropy/rdrand.c src/entropy/entropy.h src/entropy/entropy_linux.h src/entropy/entropy_mac.h src/entropy/entropy_win.h src/entropy/rdrand.h)
source_group(Entropy FILES ${SG_ENTROPY})

set(SG_HEADERS src/sqrl_internal.h src/sqrl_expert.h src/sqrl_client.h src/sqrl_server.h src/sqrl_common.h src/detect_platform.h src/utstring.h)
source_group(Headers FILES ${SG_HEADERS})

set(SRC ${SG_CLIENT_USER} ${SG_CLIENT} ${SG_SERVER} ${SG_CRYPTO} ${SG_UTIL} ${SG_ENTROPY})

add_library(sqrl_obj OBJECT ${SRC})
add_dependencies(sqrl_obj libsodium)
add_library(sqrl SHARED ${SG_HEADERS} $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(sqrl ${LIBS})

add_executable(entropy src/cli/entropy_cli.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(entropy sqrl)
set_target_properties(entropy PROPERTIES FOLDER CLI)

add_executable(enscrypt src/cli/enscrypt_bin.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(enscrypt sqrl)
set_target_properties(enscrypt PROPERTIES FOLDER CLI)

add_executable(makevectors src/cli/makevectors.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(makevectors sqrl)
set_target_properties(makevectors PROPERTIES FOLDER CLI)

add_executable(genrandom src/cli/genrandom.c)
target_link_libraries(genrandom sqrl)
set_target_properties(genrandom PROPERTIES FOLDER CLI)

add_executable(gcm_test src/crypto/gcm.c src/crypto/aes.c src/test/gcmtest.c)
add_dependencies(gcm_test libsodium)
set_target_properties(gcm_test PROPERTIES FOLDER Tests)

add_executable(storage_test src/test/storage_test.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(storage_test sqrl)
set_target_properties(storage_test PROPERTIES FOLDER Tests)

add_executable(encdec_test src/test/encdec_test.c)
target_link_libraries(encdec_test sqrl)
set_target_properties(encdec_test PROPERTIES FOLDER Tests)

add_executable(crypto_test src/test/crypto_test.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(crypto_test sqrl)
set_target_properties(crypto_test PROPERTIES FOLDER Tests)

add_executable(user_test src/test/user.c $<TARGET_OBJECTS:sqrl_obj>)
target_link_libraries(user_test sqrl)
set_target_properties(user_test PROPERTIES FOLDER Tests)

add_executable(uri_test src/test/uri_test.c)
target_link_libraries(uri_test sqrl)
set_target_properties(uri_test PROPERTIES FOLDER Tests)

add_executable(client_test src/test/client.c)
target_link_libraries(client_test sqrl)
set_target_properties(client_test PROPERTIES FOLDER Tests)

add_executable(protocol_test src/test/protocol.c)
target_link_libraries(protocol_test sqrl)
set_target_properties(protocol_test PROPERTIES FOLDER Tests)

add_executable(server_test src/test/server_test.c)
target_link_libraries(server_test sqrl)
set_target_properties(server_test PROPERTIES FOLDER Tests)

install(FILES src/utstring.h src/sqrl_expert.h src/sqrl_client.h src/sqrl_server.h src/sqrl_common.h
	DESTINATION include)

install(TARGETS sqrl LIBRARY DESTINATION lib)

#documentation
find_package(Doxygen QUIET)
if(DOXYGEN_FOUND)
	set(HEADERS 
		${CMAKE_SOURCE_DIR}/src/sqrl_common.h 
		${CMAKE_SOURCE_DIR}/src/sqrl_client.h 
		${CMAKE_SOURCE_DIR}/src/sqrl_server.h 
		${CMAKE_SOURCE_DIR}/src/sqrl_expert.h)
	set(DOCSRC ${HEADERS} ${SRC} ${CMAKE_BINARY_DIR}/html/mainpage.dox)
	string(REPLACE ";" " " DOCSRCSTR "${DOCSRC}")
	file(COPY ${CMAKE_SOURCE_DIR}/html DESTINATION ${CMAKE_BINARY_DIR})
	configure_file(${CMAKE_SOURCE_DIR}/Doxyfile.in ${CMAKE_BINARY_DIR}/Doxyfile)
	configure_file(${CMAKE_SOURCE_DIR}/html/mainpage.dox.in ${CMAKE_BINARY_DIR}/html/mainpage.dox)

	add_custom_target(doc
		${DOXYGEN_EXECUTABLE} ${CMAKE_BINARY_DIR}/Doxyfile
		WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
		COMMENT "Generating API documentation with Doxygen" VERBATIM
	)
    set_target_properties(doc PROPERTIES FOLDER meta_build)
endif(DOXYGEN_FOUND)

# Tests
include(CTest)
add_test(encdec_test ${EXECUTABLE_OUTPUT_PATH}/encdec_test)
add_test(gcm_test ${EXECUTABLE_OUTPUT_PATH}/gcm_test)
add_test(crypto_test ${EXECUTABLE_OUTPUT_PATH}/crypto_test)
add_test(uri_test ${EXECUTABLE_OUTPUT_PATH}/uri_test)
add_test(storage_test ${EXECUTABLE_OUTPUT_PATH}/storage_test)
add_test(user_test ${EXECUTABLE_OUTPUT_PATH}/user_test)
add_test(client_test ${EXECUTABLE_OUTPUT_PATH}/client_test)
add_test(server_test ${EXECUTABLE_OUTPUT_PATH}/server_test)
add_test(protocol_test ${EXECUTABLE_OUTPUT_PATH}/protocol_test)
add_test(NAME sodium-test
	WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/src/libsodium/src/libsodium
	COMMAND make check)

add_custom_target(check ${CMAKE_CTEST_COMMAND} -D Experimental)
set_target_properties(check PROPERTIES FOLDER meta_build)
