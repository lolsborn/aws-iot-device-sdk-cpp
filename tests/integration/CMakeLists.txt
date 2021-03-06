cmake_minimum_required(VERSION 3.2 FATAL_ERROR)
project(aws-iot-cpp-integration-tests CXX)

######################################
# Section : Disable in-source builds #
######################################

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
	message( FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there. You may need to remove CMakeCache.txt and CMakeFiles folder." )
endif()

########################################
# Section : Common Build setttings #
########################################
# Set required compiler standard to standard c++11. Disable extensions.
set(CMAKE_CXX_STANDARD 11) # C++11...
set(CMAKE_CXX_STANDARD_REQUIRED ON) #...is required...
set(CMAKE_CXX_EXTENSIONS OFF) #...without compiler extensions like gnu++11

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/archive)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Configure Compiler flags
if(UNIX AND NOT APPLE)
	# Prefer pthread if found
	set(THREADS_PREFER_PTHREAD_FLAG ON)
	set(CUSTOM_COMPILER_FLAGS "-fno-exceptions -Wall -Werror")
elseif(APPLE)
	set(CUSTOM_COMPILER_FLAGS "-fno-exceptions")
elseif(WIN32)
	set(CUSTOM_COMPILER_FLAGS "/W4")
endif()

####################################
# Target : Build Integration Tests #
####################################
set(INTEGRATION_TEST_TARGET_NAME aws-iot-integration-tests)
# Add Target
add_executable(${INTEGRATION_TEST_TARGET_NAME} "${CMAKE_SOURCE_DIR}/common/ConfigCommon.cpp")

target_include_directories(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${CMAKE_BINARY_DIR}/third_party/rapidjson/src/include)
target_include_directories(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${CMAKE_SOURCE_DIR}/include)

# Configure Threading library
find_package(Threads REQUIRED)
target_link_libraries(${INTEGRATION_TEST_TARGET_NAME} PUBLIC "Threads::Threads")

# Integration Tests
file(GLOB_RECURSE INTEGRATION_TEST_SOURCES FOLLOW_SYMLINKS ${CMAKE_SOURCE_DIR}/tests/integration/src/*.cpp)

# Add Target specific includes
target_include_directories(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${CMAKE_SOURCE_DIR}/common)
target_include_directories(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${CMAKE_SOURCE_DIR}/tests/integration/include)
target_sources(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${INTEGRATION_TEST_SOURCES})

# Copy Json config file
add_custom_command(TARGET ${INTEGRATION_TEST_TARGET_NAME} POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E
		copy ${CMAKE_SOURCE_DIR}/common/SampleConfig.json $<TARGET_FILE_DIR:${INTEGRATION_TEST_TARGET_NAME}>/config/IntegrationTestConfig.json)

target_link_libraries(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${THREAD_LIBRARY_LINK_STRING})
target_link_libraries(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${SDK_TARGET_NAME})
set_property(TARGET ${INTEGRATION_TEST_TARGET_NAME} APPEND_STRING PROPERTY COMPILE_FLAGS ${CUSTOM_COMPILER_FLAGS})

# Gather list of all .cert files in "/cert"
add_custom_command(TARGET ${INTEGRATION_TEST_TARGET_NAME} PRE_BUILD
		COMMAND ${CMAKE_COMMAND} -E
		copy_directory ${CMAKE_SOURCE_DIR}/certs $<TARGET_FILE_DIR:${INTEGRATION_TEST_TARGET_NAME}>/certs)

if(MSVC)
	target_sources(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${CMAKE_SOURCE_DIR}/common/ConfigCommon.hpp)
	source_group("Header Files\\Tests\\Integration" FILES ${CMAKE_SOURCE_DIR}/common/ConfigCommon.hpp)
	source_group("Source Files\\Tests\\Integration" FILES ${CMAKE_SOURCE_DIR}/common/ConfigCommon.cpp)

	file(GLOB_RECURSE INTEGRATION_TEST_HEADERS FOLLOW_SYMLINKS ${CMAKE_SOURCE_DIR}/tests/integration/include/*.hpp)
	target_sources(${INTEGRATION_TEST_TARGET_NAME} PUBLIC ${INTEGRATION_TEST_HEADERS})
	source_group("Header Files\\Tests\\Integration" FILES ${INTEGRATION_TEST_HEADERS})
	source_group("Source Files\\Tests\\Integration" FILES ${INTEGRATION_TEST_SOURCES})
endif()

#########################
# Add Network libraries #
#########################

set(NETWORK_WRAPPER_DEST_TARGET ${INTEGRATION_TEST_TARGET_NAME})
include(${CMAKE_SOURCE_DIR}/network/CMakeLists.txt.in)