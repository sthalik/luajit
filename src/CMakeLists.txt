cmake_minimum_required(VERSION 3.0)

# option(LJ_TARGET_GC64 "Enable GC64 mode for x64" OFF)
set(LUA_MULTILIB "lib" CACHE PATH "The name of lib directory." )

set(MINILUA_EXE minilua)
if ("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Windows")
  set(MINILUA_EXE minilua.exe)
endif()
set(MINILUA_PATH ${CMAKE_CURRENT_BINARY_DIR}/minilua/${MINILUA_EXE})

# Build the minilua for host platform
make_directory(${CMAKE_CURRENT_BINARY_DIR}/minilua)

add_custom_command(OUTPUT ${MINILUA_PATH}
  COMMAND ${CMAKE_COMMAND} ${CMAKE_CURRENT_SOURCE_DIR}/host/cmake/minilua
  COMMAND ${CMAKE_COMMAND} --build ${CMAKE_CURRENT_BINARY_DIR}/minilua
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/minilua)

add_custom_target(MINILUA_BUILD ALL
  DEPENDS ${MINILUA_PATH}
)

include(TestBigEndian)
TEST_BIG_ENDIAN(LJ_BIG_ENDIAN)

include(CheckTypeSize)
include(CheckCXXCompilerFlag)
check_cxx_compiler_flag(-fno-stack-protector NO_STACK_PROTECTOR_FLAG)

include(DetectArchitecture)
detect_architecture(LJ_DETECTED_ARCH)

include(DetectFPUApi)
detect_fpu_mode(LJ_DETECTED_FPU_MODE)
detect_fpu_abi(LJ_DETECTED_FPU_ABI)
message(STATUS "FPU: ${LJ_DETECTED_FPU_MODE} FPU ABI: ${LJ_DETECTED_FPU_ABI}")

find_library(LIBM NAMES m)
find_library(LIBDL NAMES dl)

set(TARGET_ARCH "")
set(DASM_FLAGS "")

set(LJ_TARGET_ARCH "")
if("${LJ_DETECTED_ARCH}" STREQUAL "x86")
  set(LJ_TARGET_ARCH "x86")
elseif("${LJ_DETECTED_ARCH}" STREQUAL "x86_64")
  set(LJ_TARGET_ARCH "x64")
elseif("${LJ_DETECTED_ARCH}" STREQUAL "AArch64")
  set(LJ_TARGET_ARCH "arm64")
  if (LJ_BIG_ENDIAN)
    set(TARGET_ARCH -D__AARCH64EB__=1)
  endif()
elseif("${LJ_DETECTED_ARCH}" STREQUAL "ARM")
  set(LJ_TARGET_ARCH "arm")
elseif("${LJ_DETECTED_ARCH}" STREQUAL "Mips64")
  set(LJ_TARGET_ARCH "mips64")
  if (NOT LJ_BIG_ENDIAN)
    set(TARGET_ARCH -D__MIPSEL__=1)
  endif()
elseif("${LJ_DETECTED_ARCH}" STREQUAL "Mips")
  set(LJ_TARGET_ARCH "mips")
  if (NOT LJ_BIG_ENDIAN)
    set(TARGET_ARCH -D__MIPSEL__=1)
  endif()
elseif("${LJ_DETECTED_ARCH}" STREQUAL "PowerPC")
  if (LJ_64)
    set(LJ_TARGET_ARCH "ppc64")
  else()
    set(LJ_TARGET_ARCH "ppc")
  endif()
  if (LJ_BIG_ENDIAN)
    set(TARGET_ARCH -DLJ_ARCH_ENDIAN=LUAJIT_BE)
  else()
    set(TARGET_ARCH -DLJ_ARCH_ENDIAN=LUAJIT_LE)
  endif()
else()
  message(FATAL_ERROR "Unsupported target architecture: '${LJ_DETECTED_ARCH}'")
endif()

if("${LJ_DETECTED_FPU_MODE}" STREQUAL "Hard")
  set(DASM_FLAGS ${DASM_FLAGS} -D FPU)
  set(TARGET_ARCH ${TARGET_ARCH} -DDLJ_ARCH_HASFPU=1)
else()
  set(TARGET_ARCH ${TARGET_ARCH} -DDLJ_ARCH_HASFPU=0)
endif()

if("${LJ_DETECTED_FPU_ABI}" STREQUAL "Hard")
  set(DASM_FLAGS ${DASM_FLAGS} -D HFABI)
  set(TARGET_ARCH ${TARGET_ARCH} -DLJ_ABI_SOFTFP=0)
else()
  set(TARGET_ARCH ${TARGET_ARCH} -DLJ_ABI_SOFTFP=1)
endif()

set(TARGET_ARCH ${TARGET_ARCH} -DLUAJIT_TARGET=LUAJIT_ARCH_${LJ_TARGET_ARCH})

if (WINDOWS)
  set(DASM_FLAGS ${DASM_FLAGS} -D WIN)
elseif(IOS)
  set(DASM_FLAGS ${DASM_FLAGS} -D IOS)
  set(DASM_FLAGS ${DASM_FLAGS} -D NO_UNWIND)
  set(TARGET_ARCH ${TARGET_ARCH} -DLJ_NO_UNWIND=1)
endif()

set(LJ_DEFINITIONS "")

set(LJ_ENABLE_LARGEFILE 1)
if (ANDROID AND (${CMAKE_SYSTEM_VERSION} LESS 21))
  set(LJ_ENABLE_LARGEFILE 0)
elseif (WINDOWS)
  set(LJ_ENABLE_LARGEFILE 0)
endif()

if (LJ_ENABLE_LARGEFILE)
  set(LJ_DEFINITIONS ${LJ_DEFINITIONS}
      -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE)
endif()

set(LJ_FFI 1)
set(LJ_JIT 1)
if (IOS)
  set(LJ_JIT 0)
endif()

if (${CMAKE_SIZEOF_VOID_P} EQUAL 8)
  set(LJ_64 1)
endif()

set(BUILDVM_ARCH_H ${CMAKE_CURRENT_BINARY_DIR}/buildvm_arch.h)
set(DASM_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../dynasm/dynasm.lua)

if (NOT LJ_BIG_ENDIAN)
  set(DASM_FLAGS ${DASM_FLAGS} -D ENDIAN_LE)
else()
  set(DASM_FLAGS ${DASM_FLAGS} -D ENDIAN_BE)
endif()

if (LJ_64)
  set(DASM_FLAGS ${DASM_FLAGS} -D P64)
endif()

if (LJ_FFI)
  set(DASM_FLAGS ${DASM_FLAGS} -D FFI)
endif()

if (LJ_JIT)
  set(DASM_FLAGS ${DASM_FLAGS} -D JIT)
endif()

set(DASM_ARCH ${LJ_TARGET_ARCH})

if ("${LJ_TARGET_ARCH}" STREQUAL "x64")
  set(DASM_ARCH "x86")
endif()

set(DASM_FLAGS ${DASM_FLAGS} -D VER=)

set(TARGET_OS_FLAGS "")
if ("${CMAKE_SYSTEM_NAME}" STREQUAL "Android")
  set(TARGET_OS_FLAGS ${TARGET_OS_FLAGS} -D LUAJIT_OS=LUAJIT_OS_LINUX)
elseif ("${CMAKE_SYSTEM_NAME}" STREQUAL "Windows")
  set(TARGET_OS_FLAGS ${TARGET_OS_FLAGS} -D LUAJIT_OS=LUAJIT_OS_WINDOWS)
elseif ("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  set(TARGET_OS_FLAGS ${TARGET_OS_FLAGS} -D LUAJIT_OS=LUAJIT_OS_OSX)
elseif ("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
  set(TARGET_OS_FLAGS ${TARGET_OS_FLAGS} -D LUAJIT_OS=LUAJIT_OS_LINUX)
else()
  set(TARGET_OS_FLAGS ${TARGET_OS_FLAGS} -D LUAJIT_OS=LUAJIT_OS_OTHER)
endif()

set(TARGET_ARCH ${TARGET_ARCH} ${TARGET_OS_FLAGS})
set(LJ_DEFINITIONS ${LJ_DEFINITIONS} ${TARGET_OS_FLAGS})

set(VM_DASC_PATH ${CMAKE_CURRENT_SOURCE_DIR}/vm_${DASM_ARCH}.dasc)

add_custom_command(OUTPUT ${BUILDVM_ARCH_H}
  COMMAND ${MINILUA_PATH} ${DASM_PATH} ${DASM_FLAGS} -o ${BUILDVM_ARCH_H} ${VM_DASC_PATH}
  DEPENDS ${MINILUA_PATH})
add_custom_target(BUILDVM_ARCH_H_BUILD ALL
  DEPENDS ${BUILDVM_ARCH_H}
)

# Build the buildvm for host platform
set(BUILDVM_COMPILER_FLAGS "${TARGET_ARCH}")

set(BUILDVM_COMPILER_FLAGS_PATH "${CMAKE_CURRENT_BINARY_DIR}/buildvm_flags.config")
file(WRITE ${BUILDVM_COMPILER_FLAGS_PATH} "${BUILDVM_COMPILER_FLAGS}")

set(BUILDVM_EXE buildvm)
if ("${CMAKE_HOST_SYSTEM_NAME}" STREQUAL "Windows")
  set(BUILDVM_EXE buildvm.exe)
endif()
set(BUILDVM_PATH ${CMAKE_CURRENT_BINARY_DIR}/buildvm/${BUILDVM_EXE})

make_directory(${CMAKE_CURRENT_BINARY_DIR}/buildvm)

add_custom_command(OUTPUT ${BUILDVM_PATH}
  COMMAND ${CMAKE_COMMAND} ${CMAKE_CURRENT_SOURCE_DIR}/host/cmake/buildvm -DEXTRA_COMPILER_FLAGS_FILE=${BUILDVM_COMPILER_FLAGS_PATH}
  COMMAND ${CMAKE_COMMAND} --build ${CMAKE_CURRENT_BINARY_DIR}/buildvm
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/host/cmake/buildvm/CMakeLists.txt
  DEPENDS BUILDVM_ARCH_H_BUILD
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/buildvm)

add_custom_target(BUILDVM_BUILD ALL
  DEPENDS ${BUILDVM_PATH}
)

set(LJVM_MODE elfasm)
if (APPLE)
  set(LJVM_MODE machasm)
elseif(WINDOWS)
  set(LJVM_MODE peobj)
endif()

set(LJ_VM_S_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_vm.S)
add_custom_command(OUTPUT ${LJ_VM_S_PATH}
  COMMAND ${BUILDVM_PATH} -m ${LJVM_MODE} -o ${LJ_VM_S_PATH}
  DEPENDS ${BUILDVM_PATH}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/)

add_custom_target(LJ_VM_S_BUILD ALL
  DEPENDS ${LJ_VM_S_BUILD}
)

make_directory(${CMAKE_CURRENT_BINARY_DIR}/jit)
set(LJ_LIBDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_libdef.h)
set(LJ_RECDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_recdef.h)
set(LJ_FFDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_ffdef.h)
set(LJ_BCDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_bcdef.h)
set(LJ_VMDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/jit/vmdef.lua)

set(LJ_LIB_SOURCES
  lib_base.c lib_math.c lib_bit.c lib_string.c lib_table.c
  lib_io.c lib_os.c lib_package.c lib_debug.c lib_jit.c lib_ffi.c)
add_custom_command(
  OUTPUT ${LJ_LIBDEF_PATH} ${LJ_VMDEF_PATH} ${LJ_RECDEF_PATH} ${LJ_FFDEF_PATH}
  COMMAND ${BUILDVM_PATH} -m libdef -o ${LJ_LIBDEF_PATH} ${LJ_LIB_SOURCES}
  COMMAND ${BUILDVM_PATH} -m recdef -o ${LJ_RECDEF_PATH} ${LJ_LIB_SOURCES}
  COMMAND ${BUILDVM_PATH} -m ffdef -o ${LJ_FFDEF_PATH} ${LJ_LIB_SOURCES}
  COMMAND ${BUILDVM_PATH} -m bcdef -o ${LJ_BCDEF_PATH} ${LJ_LIB_SOURCES}
  COMMAND ${BUILDVM_PATH} -m vmdef -o ${LJ_VMDEF_PATH} ${LJ_LIB_SOURCES}
  DEPENDS ${BUILDVM_PATH} ${LJ_LIB_SOURCE}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/)

add_custom_target(LJ_DEF_BUILD ALL
  DEPENDS ${LJ_LIBDEF_PATH} ${LJ_RECDEF_PATH} ${LJ_VMDEF_PATH}
  DEPENDS ${LJ_FFDEF_PATH} ${LJ_BCDEF_PATH}
)

set(LJ_FOLDDEF_PATH ${CMAKE_CURRENT_BINARY_DIR}/lj_folddef.h)

set(LJ_FOLDDEF_SOURCE lj_opt_fold.c)
add_custom_command(
  OUTPUT ${LJ_FOLDDEF_PATH}
  COMMAND ${BUILDVM_PATH} -m folddef -o ${LJ_FOLDDEF_PATH} ${LJ_FOLDDEF_SOURCE}
  DEPENDS ${BUILDVM_PATH}
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/)

add_custom_target(LJ_FOLDDEF_BUILD ALL
  DEPENDS ${LJ_FOLDDEF_PATH}
)

set(luajit_sources lib_aux.c
    lib_base.c
    lib_bit.c
    lib_debug.c
    lib_ffi.c
    lib_init.c
    lib_io.c
    lib_jit.c
    lib_math.c
    lib_os.c
    lib_package.c
    lib_string.c
    lib_table.c
    lj_alloc.c
    lj_api.c
    lj_asm.c
    lj_bc.c
    lj_bcread.c
    lj_bcwrite.c
    lj_buf.c
    lj_carith.c
    lj_ccall.c
    lj_ccallback.c
    lj_cconv.c
    lj_cdata.c
    lj_char.c
    lj_clib.c
    lj_cparse.c
    lj_crecord.c
    lj_ctype.c
    lj_debug.c
    lj_dispatch.c
    lj_err.c
    lj_ffrecord.c
    lj_func.c
    lj_gc.c
    lj_gdbjit.c
    lj_ir.c
    lj_lex.c
    lj_lib.c
    lj_load.c
    lj_mcode.c
    lj_meta.c
    lj_obj.c
    lj_opt_dce.c
    lj_opt_fold.c
    lj_opt_loop.c
    lj_opt_mem.c
    lj_opt_narrow.c
    lj_opt_sink.c
    lj_opt_split.c
    lj_parse.c
    lj_profile.c
    lj_record.c
    lj_snap.c
    lj_state.c
    lj_str.c
    lj_strfmt.c
    lj_strfmt_num.c
    lj_strscan.c
    lj_tab.c
    lj_trace.c
    lj_udata.c
    lj_vmevent.c
    lj_vmmath.c
    lj_vm.S)

# Build the luajit static library
add_library(libluajit ${luajit_sources})
set_target_properties(libluajit PROPERTIES OUTPUT_NAME luajit)
add_dependencies(libluajit
  BUILDVM_ARCH_H_BUILD
  BUILDVM_BUILD
  LJ_DEF_BUILD
  LJ_FOLDDEF_BUILD)
target_include_directories(libluajit PRIVATE
  ${CMAKE_CURRENT_BINARY_DIR}
  ${CMAKE_CURRENT_SOURCE_DIR})
if (BUILD_SHARED_LIBS)
  set(LJ_DEFINITIONS ${LJ_DEFINITIONS} -DLUA_BUILD_AS_DLL)
  target_link_libraries("-image_base 7fff04c4a000")
endif()
if(UNIX)
  target_link_libraries(libluajit ${LIBM} ${LIBDL})
endif(UNIX)

set(LJ_DEFINITIONS ${LJ_DEFINITIONS} -DLUA_MULTILIB="${LUA_MULTILIB}")
target_compile_definitions(libluajit PRIVATE ${LJ_DEFINITIONS})

if("${LJ_TARGET_ARCH}" STREQUAL "x86")
  if (CMAKE_COMPILER_IS_CLANGXX OR CMAKE_COMPILER_IS_GNUCXX)
    target_compile_definitions(libluajit -march=i686 -msse -msse2 -mfpmath=sse)
  endif()
endif()

set(LJ_COMPILE_OPTIONS -U_FORTIFY_SOURCE)
if (NO_STACK_PROTECTOR_FLAG)
  set(LJ_COMPILE_OPTIONS ${LJ_COMPILE_OPTIONS} -fno-stack-protector)
endif()
if (IOS AND ("${LJ_TARGET_ARCH}" STREQUAL "arm64"))
  set(LJ_COMPILE_OPTIONS ${LJ_COMPILE_OPTIONS} -fno-omit-frame-pointer)
endif()

target_compile_options(libluajit PRIVATE ${LJ_COMPILE_OPTIONS})

# Build the luajit binary
add_executable(luajit luajit.c)
target_link_libraries(luajit libluajit)
if (APPLE AND LJ_64)
  target_link_libraries(luajit "-pagezero_size 10000" "-image_base 100000000")
endif()
target_compile_definitions(luajit PRIVATE ${LJ_DEFINITIONS})

set(luajit_headers lauxlib.h lua.h luaconf.h luajit.h lualib.h)
install(FILES ${luajit_headers} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/luajit)
install(TARGETS libluajit
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(TARGETS luajit DESTINATION "${CMAKE_INSTALL_BINDIR}")
