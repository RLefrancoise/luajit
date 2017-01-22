project (luajit)
cmake_minimum_required (VERSION 3.2.0)

FUNCTION(PREPEND var prefix)
   SET(listVar "")
   FOREACH(f ${ARGN})
      LIST(APPEND listVar "${prefix}/${f}")
   ENDFOREACH(f)
   SET(${var} "${listVar}" PARENT_SCOPE)
ENDFUNCTION(PREPEND)

include(TestBigEndian)

TEST_BIG_ENDIAN(HOST_BIG_ENDIAN)

set(LUAJIT_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dependencies/luajit-2.1)
set(LUAJIT_SRC_DIR ${LUAJIT_DIR}/src)
set(LUAJIT_HOST_DIR ${LUAJIT_SRC_DIR}/host)
set(LUAJIT_DYNASM_DIR ${LUAJIT_DIR}/dynasm)

set(LJ_NAME lua51)
set(LJDLLNAME ${LJ_NAME}.dll)
set(LJLIBNAME ${LJ_NAME}.lib)

set(ALL_LIB
	lib_base.c lib_math.c lib_bit.c lib_string.c lib_table.c lib_io.c lib_os.c lib_package.c lib_debug.c lib_jit.c lib_ffi.c
)
PREPEND(ALL_LIB_PATHS ${LUAJIT_SRC_DIR} ${ALL_LIB})

set(LUAJIT_OUT_HOST_DIR ${CMAKE_CURRENT_BINARY_DIR}/host)

set(LUAJIT_ENABLE_LUA52COMPAT ON CACHE BOOL "Enable Lua 5.2 Compatibility mode")
set(LUAJIT_ENABLE_GC64 ON CACHE BOOL "Enable new GC64")

if (LUAJIT_ENABLE_LUA52COMPAT)
	add_definitions(-DLUAJIT_ENABLE_LUA52COMPAT)
endif()


if (LUAJIT_ENABLE_GC64)
	add_definitions(-DLUAJIT_ENABLE_GC64)
endif()

set(DYNASM_FLAGS
	-D JIT
	-D FFI
)

# Target Architecture

if (CMAKE_CL_64)
	set(LJARCH x64)
else()
	set(LJARCH x86)
endif()

if (LJARCH STREQUAL x64)
	set(DASM_ARCH x64)
elseif (LJARCH STREQUAL x86)
	set(DASM_ARCH x86)
elseif (LJARCH STREQUAL arm)
	set(DASM_ARCH arm)
elseif (LJARCH STREQUAL arm64)
	set(DASM_ARCH arm64)
elseif (LJARCH STREQUAL mips)
	set(DASM_ARCH mips)
elseif (LJARCH STREQUAL ppc)
	set(DASM_ARCH ppc)
endif()

if (HOST_BIG_ENDIAN)
	list(APPEND DYNASM_FLAGS
		-D ENDIAN_BE
	)
else()
	list(APPEND DYNASM_FLAGS
		-D ENDIAN_LE
	)
endif()

### MINILUA ###

set(minilua_sources
	${LUAJIT_HOST_DIR}/minilua.c
)

include_directories(
	${LUAJIT_SRC_DIR}
	${LUAJIT_HOST_DIR}
	${LUAJIT_OUT_HOST_DIR}
)

if(WIN32)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif()

add_executable(minilua
	${minilua_sources}
)

### DYNASM ###

if(WIN32)
	list(APPEND DYNASM_FLAGS
		-D WIN
	)
endif()

if (CMAKE_CL_64)
	list(APPEND DYNASM_FLAGS
		-D P64
	)
endif()

# BUILDVM_ARCH

set(BUILDVM_ARCH_H
	${LUAJIT_OUT_HOST_DIR}/buildvm_arch.h
)

message("Using vm_${DASM_ARCH}.dasc with architexture ${LJARCH}")

add_custom_command(OUTPUT ${BUILDVM_ARCH_H}
	COMMAND $<TARGET_FILE:minilua> ${LUAJIT_DYNASM_DIR}/dynasm.lua -LN ${DYNASM_FLAGS} -o ${BUILDVM_ARCH_H} ${LUAJIT_SRC_DIR}/vm_${DASM_ARCH}.dasc
	DEPENDS minilua
)

add_custom_target(PREPARE_BUILDVM_ARCH
	DEPENDS ${BUILDVM_ARCH_H}
)

# BUILDVM

set(buildvm_sources
	${LUAJIT_HOST_DIR}/buildvm.c
	${LUAJIT_HOST_DIR}/buildvm_asm.c
	${LUAJIT_HOST_DIR}/buildvm_fold.c
	${LUAJIT_HOST_DIR}/buildvm_lib.c
	${LUAJIT_HOST_DIR}/buildvm_peobj.c
	${LUAJIT_HOST_DIR}/buildvm.h
)

if (NOT WIN32)
	list(APPEND buildvm_sources
		${LUAJIT_HOST_DIR}/buildvm_libbc.h
	)
endif()

add_executable(buildvm
	${buildvm_sources}
)

add_dependencies(buildvm
	PREPARE_BUILDVM_ARCH
)

set_target_properties(buildvm PROPERTIES
	PREFIX ""
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/bin"
)

# VM

add_custom_command(OUTPUT lj_vm.obj
	COMMAND $<TARGET_FILE:buildvm> -m peobj -o lj_vm.obj
)

add_custom_command(OUTPUT lj_bcdef.h
	COMMAND $<TARGET_FILE:buildvm> -m bcdef -o lj_bcdef.h ${ALL_LIB_PATHS}
)

add_custom_command(OUTPUT lj_ffdef.h
	COMMAND $<TARGET_FILE:buildvm> -m ffdef -o lj_ffdef.h ${ALL_LIB_PATHS}
)

add_custom_command(OUTPUT lj_libdef.h
	COMMAND $<TARGET_FILE:buildvm> -m libdef -o lj_libdef.h ${ALL_LIB_PATHS}
)

add_custom_command(OUTPUT lj_recdef.h
	COMMAND $<TARGET_FILE:buildvm> -m recdef -o lj_recdef.h ${ALL_LIB_PATHS}
)

add_custom_command(OUTPUT jit/vmdef.lua
	COMMAND $<TARGET_FILE:buildvm> -m vmdef -o jit/vmdef.lua ${ALL_LIB_PATHS}
)

add_custom_command(OUTPUT lj_folddef.h
	COMMAND $<TARGET_FILE:buildvm> -m folddef -o lj_folddef.h ${LUAJIT_SRC_DIR}/lj_opt_fold.c
)

add_custom_target(PREPARE_VM
	DEPENDS lj_vm.obj lj_bcdef.h lj_ffdef.h lj_libdef.h lj_recdef.h jit/vmdef.lua lj_folddef.h
		
)

include_directories(
	${CMAKE_CURRENT_BINARY_DIR}
)

# LUAJIT_DLL

set(luajit_dll_sources
	${LUAJIT_SRC_DIR}/ljamalg.c
)

add_library(luajit_dll SHARED
	${luajit_dll_sources}
	lj_vm.obj
)

add_dependencies(luajit_dll
	PREPARE_VM
)

set_target_properties(luajit_dll PROPERTIES
	PREFIX ""
	OUTPUT_NAME ${LJ_NAME}
	COMPILE_FLAGS "-DLUA_BUILD_AS_DLL"
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/bin"
)

# LUAJIT

set(luajit_sources
	${LUAJIT_SRC_DIR}/luajit.c
)

add_executable(luajit
	${luajit_sources}
)

target_link_libraries (luajit
	luajit_dll
)

set_target_properties(luajit PROPERTIES
	PREFIX ""
	OUTPUT_NAME luajit
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/lib"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${LJARCH}/bin"
)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
	build
)

