cmake_minimum_required(VERSION 3.25)
project(LuaJIT LANGUAGES C ASM)

set(LUAJIT_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")

# 1. Detect Architecture for DASM
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(LUAJIT_ARCH x64)
else()
    set(LUAJIT_ARCH x86)
endif()

# 2. Determine BuildVM Mode and Extension
if(WIN32)
    set(LUAJIT_VM_MODE peobj)
    set(LUAJIT_VM_EXT  obj) # Binary object file
else()
    if(APPLE)
        set(LUAJIT_VM_MODE machasm)
    else()
        set(LUAJIT_VM_MODE elfasm)
    endif()
    set(LUAJIT_VM_EXT S)    # Text assembly file
endif()

set(LUAJIT_VM_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/lj_vm.${LUAJIT_VM_EXT}")
set(LUAJIT_BUILDVM_DASC "${LUAJIT_SRC_DIR}/vm_x86.dasc") # DASM uses x86 for both 32/64 usually

# 3. Build the BuildVM Tool
add_executable(luaJIT_buildvm
    "${LUAJIT_SRC_DIR}/host/buildvm.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_asm.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_peobj.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_lib.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_fold.c"
)
target_include_directories(luaJIT_buildvm PRIVATE "${LUAJIT_SRC_DIR}")
if(MSVC OR (WIN32 AND CMAKE_C_COMPILER_ID MATCHES "Clang"))
    target_compile_definitions(luaJIT_buildvm PRIVATE _CRT_SECURE_NO_DEPRECATE)
endif()

if(USE_SANITIZERS)
    if(MSVC)
        target_compile_options(luaJIT_buildvm PRIVATE /fsanitize=address /fsanitize-address-vcasan-) # Disable
    else()
        target_compile_options(luaJIT_buildvm PRIVATE -fno-sanitize=all)
        target_link_options(luaJIT_buildvm PRIVATE -fno-sanitize=all)
    endif()
endif()

# 4. Generate the VM code
add_custom_command(
    OUTPUT "${LUAJIT_VM_OUTPUT}"
    COMMAND $<TARGET_FILE:luaJIT_buildvm> -m ${LUAJIT_VM_MODE} -o "${LUAJIT_VM_OUTPUT}" "${LUAJIT_BUILDVM_DASC}"
    DEPENDS luaJIT_buildvm "${LUAJIT_BUILDVM_DASC}"
    WORKING_DIRECTORY "${LUAJIT_SRC_DIR}"
    COMMENT "Generating LuaJIT VM: ${LUAJIT_VM_OUTPUT}"
    VERBATIM
)

# 5. Core Sources
set(LUAJIT_SOURCES
    "${LUAJIT_SRC_DIR}/lib_base.c" "${LUAJIT_SRC_DIR}/lib_math.c" "${LUAJIT_SRC_DIR}/lib_bit.c"
    "${LUAJIT_SRC_DIR}/lib_string.c" "${LUAJIT_SRC_DIR}/lib_table.c" "${LUAJIT_SRC_DIR}/lib_io.c"
    "${LUAJIT_SRC_DIR}/lib_os.c" "${LUAJIT_SRC_DIR}/lib_package.c" "${LUAJIT_SRC_DIR}/lib_debug.c"
    "${LUAJIT_SRC_DIR}/lib_jit.c" "${LUAJIT_SRC_DIR}/lib_ffi.c" "${LUAJIT_SRC_DIR}/lib_buffer.c"
    "${LUAJIT_SRC_DIR}/lj_assert.c" "${LUAJIT_SRC_DIR}/lj_gc.c" "${LUAJIT_SRC_DIR}/lj_err.c"
    "${LUAJIT_SRC_DIR}/lj_char.c" "${LUAJIT_SRC_DIR}/lj_bc.c" "${LUAJIT_SRC_DIR}/lj_obj.c"
    "${LUAJIT_SRC_DIR}/lj_buf.c" "${LUAJIT_SRC_DIR}/lj_str.c" "${LUAJIT_SRC_DIR}/lj_tab.c"
    "${LUAJIT_SRC_DIR}/lj_func.c" "${LUAJIT_SRC_DIR}/lj_udata.c" "${LUAJIT_SRC_DIR}/lj_meta.c"
    "${LUAJIT_SRC_DIR}/lj_debug.c" "${LUAJIT_SRC_DIR}/lj_prng.c" "${LUAJIT_SRC_DIR}/lj_state.c"
    "${LUAJIT_SRC_DIR}/lj_dispatch.c" "${LUAJIT_SRC_DIR}/lj_vmevent.c" "${LUAJIT_SRC_DIR}/lj_vmmath.c"
    "${LUAJIT_SRC_DIR}/lj_strscan.c" "${LUAJIT_SRC_DIR}/lj_strfmt.c" "${LUAJIT_SRC_DIR}/lj_strfmt_num.c"
    "${LUAJIT_SRC_DIR}/lj_serialize.c" "${LUAJIT_SRC_DIR}/lj_api.c" "${LUAJIT_SRC_DIR}/lj_profile.c"
    "${LUAJIT_SRC_DIR}/lj_lex.c" "${LUAJIT_SRC_DIR}/lj_parse.c" "${LUAJIT_SRC_DIR}/lj_bcread.c"
    "${LUAJIT_SRC_DIR}/lj_bcwrite.c" "${LUAJIT_SRC_DIR}/lj_load.c" "${LUAJIT_SRC_DIR}/lj_ir.c"
    "${LUAJIT_SRC_DIR}/lj_opt_mem.c" "${LUAJIT_SRC_DIR}/lj_opt_fold.c" "${LUAJIT_SRC_DIR}/lj_opt_narrow.c"
    "${LUAJIT_SRC_DIR}/lj_opt_dce.c" "${LUAJIT_SRC_DIR}/lj_opt_loop.c" "${LUAJIT_SRC_DIR}/lj_opt_split.c"
    "${LUAJIT_SRC_DIR}/lj_opt_sink.c" "${LUAJIT_SRC_DIR}/lj_mcode.c" "${LUAJIT_SRC_DIR}/lj_snap.c"
    "${LUAJIT_SRC_DIR}/lj_record.c" "${LUAJIT_SRC_DIR}/lj_crecord.c" "${LUAJIT_SRC_DIR}/lj_ffrecord.c"
    "${LUAJIT_SRC_DIR}/lj_asm.c" "${LUAJIT_SRC_DIR}/lj_trace.c" "${LUAJIT_SRC_DIR}/lj_gdbjit.c"
    "${LUAJIT_SRC_DIR}/lj_ctype.c" "${LUAJIT_SRC_DIR}/lj_cdata.c" "${LUAJIT_SRC_DIR}/lj_cconv.c"
    "${LUAJIT_SRC_DIR}/lj_ccall.c" "${LUAJIT_SRC_DIR}/lj_ccallback.c" "${LUAJIT_SRC_DIR}/lj_carith.c"
    "${LUAJIT_SRC_DIR}/lj_clib.c" "${LUAJIT_SRC_DIR}/lj_cparse.c" "${LUAJIT_SRC_DIR}/lj_lib.c"
    "${LUAJIT_SRC_DIR}/lj_alloc.c" "${LUAJIT_SRC_DIR}/lib_aux.c" "${LUAJIT_SRC_DIR}/lib_init.c"
)

# 6. Final Library
add_library(LuaJIT STATIC ${LUAJIT_SOURCES} "${LUAJIT_VM_OUTPUT}")
target_include_directories(LuaJIT PUBLIC "${LUAJIT_SRC_DIR}")

if(WIN32)
    target_compile_definitions(LuaJIT PRIVATE LUA_BUILD_AS_DLL) # Allows FFI to work
endif()

if(NOT MSVC)
    target_compile_options(LuaJIT PRIVATE -U_FORTIFY_SOURCE)
endif()