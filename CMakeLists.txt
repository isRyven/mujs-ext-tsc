cmake_minimum_required(VERSION 3.12)
project(mujs LANGUAGES C)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -D_GNU_SOURCE -pedantic -Wall -Wextra -Wno-unused-parameter -fvisibility=hidden")
	if(CMAKE_C_COMPILER_ID MATCHES "Clang")
        set(CMAKE_C_FLAGS  "${CMAKE_C_FLAGS} -Wunreachable-code")
    endif()
    if (SANITIZE_ADDRESS)
        message(STATUS "Address sanitization is enabled")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -fno-omit-frame-pointer -fsanitize=address")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    endif()
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -s -ffast-math -O3")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -ggdb")
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fPIC")
endif()

include(cmake/CMakeZRC.cmake)

set(MUJS_REPL ON CACHE BOOL "mujs repl")

add_subdirectory(deps/mujs)

# increase limits
target_compile_definitions(mujs 
	PRIVATE JS_STACKSIZE=4096 JS_ENVLIMIT=512 JS_INSTRUCTION=int
)

zrc_add_resource_library(js_assets 
	WORKING_DIR ${CMAKE_CURRENT_SOURCE_DIR}/js
	manifest.json
)

file(GLOB js_scripts "${PROJECT_SOURCE_DIR}/js/*.js")
set(js_assets_temp "${CMAKE_CURRENT_BINARY_DIR}/__js_assets_temp")
file(MAKE_DIRECTORY ${js_assets_temp})
list(TRANSFORM js_scripts REPLACE "^${PROJECT_SOURCE_DIR}/js/(.+)" "\\1bin" OUTPUT_VARIABLE js_bin_scripts)

zrc_add_resources(js_assets 
	WORKING_DIR ${js_assets_temp}
	${js_bin_scripts}
)

add_custom_target(js_precompile_scripts
  	COMMAND mujs_repl -c ${js_scripts}
  	DEPENDS mujs_repl ${js_scripts}
  	WORKING_DIRECTORY ${js_assets_temp}
  	COMMENT "Precompiling assets scripts"
)

add_dependencies(js_assets js_precompile_scripts)

add_executable(tsc main.c)
target_link_libraries(tsc mujs js_assets)
