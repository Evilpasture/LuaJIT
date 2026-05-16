cmake_minimum_required(VERSION 3.25)
project(LuaJIT LANGUAGES C ASM)

set(LUAJIT_SRC_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")
set(DASM_DIR       "${CMAKE_CURRENT_SOURCE_DIR}/dynasm")

# 1. Architecture detection — mirrors the .bat logic
if(CMAKE_SYSTEM_PROCESSOR MATCHES "ARM64|aarch64")
    set(LUAJIT_DASC    "${LUAJIT_SRC_DIR}/vm_arm64.dasc")
    set(DASM_FLAGS     -D FFI -D ENDIAN_LE -D FPU -D P64)
    if(WIN32)
        list(APPEND DASM_FLAGS -D WIN)
    endif()
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(LUAJIT_DASC    "${LUAJIT_SRC_DIR}/vm_x64.dasc")
    set(DASM_FLAGS     -D JIT -D FFI -D ENDIAN_LE -D FPU -D P64)
    if(WIN32)
        list(PREPEND DASM_FLAGS -D WIN)
    endif()
else()
    set(LUAJIT_DASC    "${LUAJIT_SRC_DIR}/vm_x86.dasc")
    set(DASM_FLAGS     -D JIT -D FFI -D ENDIAN_LE -D FPU)
    if(WIN32)
        list(PREPEND DASM_FLAGS -D WIN)
    endif()
endif()

# 2. BuildVM output mode
if(WIN32)
    set(LUAJIT_VM_MODE peobj)
    set(LUAJIT_VM_EXT  obj)
else()
    if(APPLE)
        set(LUAJIT_VM_MODE machasm)
    else()
        set(LUAJIT_VM_MODE elfasm)
    endif()
    set(LUAJIT_VM_EXT S)
endif()

set(LUAJIT_VM_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/lj_vm.${LUAJIT_VM_EXT}")
set(DASM_ARCH_H      "${CMAKE_CURRENT_BINARY_DIR}/buildvm_arch.h")

# 3. Generate luajit.h from luajit_rolling.h if not present
if(NOT EXISTS "${LUAJIT_SRC_DIR}/luajit.h")
    configure_file(
        "${LUAJIT_SRC_DIR}/luajit_rolling.h"
        "${CMAKE_CURRENT_BINARY_DIR}/luajit.h"
        COPYONLY
    )
endif()

# 4. minilua — stripped-down Lua interpreter to run DynASM
add_executable(minilua "${LUAJIT_SRC_DIR}/host/minilua.c")
if(UNIX)
    target_link_libraries(minilua PRIVATE m)
endif()
if(USE_SANITIZERS AND NOT MSVC)
    target_compile_options(minilua PRIVATE -fno-sanitize=all)
    target_link_options(minilua PRIVATE -fno-sanitize=all)
endif()

# 5. genversion — generate luajit_relver.txt + luajit.h version info
#    The .bat does: git show -s --format=%ct > luajit_relver.txt
#                   minilua host/genversion.lua
set(LUAJIT_RELVER_TXT "${LUAJIT_SRC_DIR}/luajit_relver.txt")
add_custom_command(
    OUTPUT "${LUAJIT_RELVER_TXT}"
    COMMAND git show -s "--format=%ct" > "${LUAJIT_RELVER_TXT}"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    COMMENT "LuaJIT: extracting git version timestamp"
    VERBATIM
)
add_custom_command(
    OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/luajit_version_done"
    COMMAND $<TARGET_FILE:minilua> "${LUAJIT_SRC_DIR}/host/genversion.lua"
    COMMAND ${CMAKE_COMMAND} -E touch "${CMAKE_CURRENT_BINARY_DIR}/luajit_version_done"
    DEPENDS minilua "${LUAJIT_RELVER_TXT}" "${LUAJIT_SRC_DIR}/host/genversion.lua"
    WORKING_DIRECTORY "${LUAJIT_SRC_DIR}"
    COMMENT "LuaJIT: generating version header"
    VERBATIM
)
add_custom_target(luaJIT_version DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/luajit_version_done")

# 6. DynASM — generate buildvm_arch.h
add_custom_command(
    OUTPUT "${DASM_ARCH_H}"
    COMMAND $<TARGET_FILE:minilua>
            "${DASM_DIR}/dynasm.lua"
            -LN ${DASM_FLAGS}
            -o "${DASM_ARCH_H}"
            "${LUAJIT_DASC}"
    DEPENDS minilua "${LUAJIT_DASC}" "${DASM_DIR}/dynasm.lua"
    COMMENT "DynASM: generating buildvm_arch.h"
    VERBATIM
)
add_custom_target(luaJIT_dasm DEPENDS "${DASM_ARCH_H}")

# 7. buildvm host tool
add_executable(luaJIT_buildvm
    "${LUAJIT_SRC_DIR}/host/buildvm.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_asm.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_peobj.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_lib.c"
    "${LUAJIT_SRC_DIR}/host/buildvm_fold.c"
)
target_include_directories(luaJIT_buildvm PRIVATE
    "${LUAJIT_SRC_DIR}"
    "${DASM_DIR}"
    "${CMAKE_CURRENT_BINARY_DIR}"  # buildvm_arch.h + luajit.h
)
add_dependencies(luaJIT_buildvm luaJIT_dasm luaJIT_version)
if(MSVC OR (WIN32 AND CMAKE_C_COMPILER_ID MATCHES "Clang"))
    target_compile_definitions(luaJIT_buildvm PRIVATE _CRT_SECURE_NO_DEPRECATE)
endif()
if(USE_SANITIZERS)
    if(MSVC)
        target_compile_options(luaJIT_buildvm PRIVATE /fsanitize=address /fsanitize-address-vcasan-)
    else()
        target_compile_options(luaJIT_buildvm PRIVATE -fno-sanitize=all)
        target_link_options(luaJIT_buildvm PRIVATE -fno-sanitize=all)
    endif()
endif()

# 8. VM object/asm
add_custom_command(
    OUTPUT "${LUAJIT_VM_OUTPUT}"
    COMMAND $<TARGET_FILE:luaJIT_buildvm> -m ${LUAJIT_VM_MODE} -o "${LUAJIT_VM_OUTPUT}" "${LUAJIT_DASC}"
    DEPENDS luaJIT_buildvm "${LUAJIT_DASC}"
    WORKING_DIRECTORY "${LUAJIT_SRC_DIR}"
    COMMENT "BuildVM: generating lj_vm"
    VERBATIM
)

# 9. Generated headers — note folddef only takes lj_opt_fold.c
file(GLOB BUILDVM_LIB_SOURCES "${LUAJIT_SRC_DIR}/lib_*.c")

set(BUILDVM_HEADER_OUTPUTS "")
foreach(hdr lj_bcdef.h lj_ffdef.h lj_libdef.h lj_recdef.h lj_folddef.h)
    set(hdr_out "${CMAKE_CURRENT_BINARY_DIR}/${hdr}")
    list(APPEND BUILDVM_HEADER_OUTPUTS "${hdr_out}")

    if(hdr STREQUAL "lj_folddef.h")
        set(bvm_sources "${LUAJIT_SRC_DIR}/lj_opt_fold.c")  # only this one
    else()
        set(bvm_sources ${BUILDVM_LIB_SOURCES})
    endif()

    string(REPLACE ".h" "" bvm_mode "${hdr}")
    string(REPLACE "lj_" ""  bvm_mode "${bvm_mode}")

    add_custom_command(
        OUTPUT "${hdr_out}"
        COMMAND $<TARGET_FILE:luaJIT_buildvm> -m ${bvm_mode} -o "${hdr_out}" ${bvm_sources}
        DEPENDS luaJIT_buildvm ${bvm_sources}
        COMMENT "BuildVM: generating ${hdr}"
        VERBATIM
    )
endforeach()
add_custom_target(luaJIT_headers DEPENDS ${BUILDVM_HEADER_OUTPUTS})

# 10. Core sources
set(LUAJIT_SOURCES
    "${LUAJIT_SRC_DIR}/lib_base.c"     "${LUAJIT_SRC_DIR}/lib_math.c"     "${LUAJIT_SRC_DIR}/lib_bit.c"
    "${LUAJIT_SRC_DIR}/lib_string.c"   "${LUAJIT_SRC_DIR}/lib_table.c"    "${LUAJIT_SRC_DIR}/lib_io.c"
    "${LUAJIT_SRC_DIR}/lib_os.c"       "${LUAJIT_SRC_DIR}/lib_package.c"  "${LUAJIT_SRC_DIR}/lib_debug.c"
    "${LUAJIT_SRC_DIR}/lib_jit.c"      "${LUAJIT_SRC_DIR}/lib_ffi.c"      "${LUAJIT_SRC_DIR}/lib_buffer.c"
    "${LUAJIT_SRC_DIR}/lj_assert.c"    "${LUAJIT_SRC_DIR}/lj_gc.c"        "${LUAJIT_SRC_DIR}/lj_err.c"
    "${LUAJIT_SRC_DIR}/lj_char.c"      "${LUAJIT_SRC_DIR}/lj_bc.c"        "${LUAJIT_SRC_DIR}/lj_obj.c"
    "${LUAJIT_SRC_DIR}/lj_buf.c"       "${LUAJIT_SRC_DIR}/lj_str.c"       "${LUAJIT_SRC_DIR}/lj_tab.c"
    "${LUAJIT_SRC_DIR}/lj_func.c"      "${LUAJIT_SRC_DIR}/lj_udata.c"     "${LUAJIT_SRC_DIR}/lj_meta.c"
    "${LUAJIT_SRC_DIR}/lj_debug.c"     "${LUAJIT_SRC_DIR}/lj_prng.c"      "${LUAJIT_SRC_DIR}/lj_state.c"
    "${LUAJIT_SRC_DIR}/lj_dispatch.c"  "${LUAJIT_SRC_DIR}/lj_vmevent.c"   "${LUAJIT_SRC_DIR}/lj_vmmath.c"
    "${LUAJIT_SRC_DIR}/lj_strscan.c"   "${LUAJIT_SRC_DIR}/lj_strfmt.c"    "${LUAJIT_SRC_DIR}/lj_strfmt_num.c"
    "${LUAJIT_SRC_DIR}/lj_serialize.c" "${LUAJIT_SRC_DIR}/lj_api.c"       "${LUAJIT_SRC_DIR}/lj_profile.c"
    "${LUAJIT_SRC_DIR}/lj_lex.c"       "${LUAJIT_SRC_DIR}/lj_parse.c"     "${LUAJIT_SRC_DIR}/lj_bcread.c"
    "${LUAJIT_SRC_DIR}/lj_bcwrite.c"   "${LUAJIT_SRC_DIR}/lj_load.c"      "${LUAJIT_SRC_DIR}/lj_ir.c"
    "${LUAJIT_SRC_DIR}/lj_opt_mem.c"   "${LUAJIT_SRC_DIR}/lj_opt_fold.c"  "${LUAJIT_SRC_DIR}/lj_opt_narrow.c"
    "${LUAJIT_SRC_DIR}/lj_opt_dce.c"   "${LUAJIT_SRC_DIR}/lj_opt_loop.c"  "${LUAJIT_SRC_DIR}/lj_opt_split.c"
    "${LUAJIT_SRC_DIR}/lj_opt_sink.c"  "${LUAJIT_SRC_DIR}/lj_mcode.c"     "${LUAJIT_SRC_DIR}/lj_snap.c"
    "${LUAJIT_SRC_DIR}/lj_record.c"    "${LUAJIT_SRC_DIR}/lj_crecord.c"   "${LUAJIT_SRC_DIR}/lj_ffrecord.c"
    "${LUAJIT_SRC_DIR}/lj_asm.c"       "${LUAJIT_SRC_DIR}/lj_trace.c"     "${LUAJIT_SRC_DIR}/lj_gdbjit.c"
    "${LUAJIT_SRC_DIR}/lj_ctype.c"     "${LUAJIT_SRC_DIR}/lj_cdata.c"     "${LUAJIT_SRC_DIR}/lj_cconv.c"
    "${LUAJIT_SRC_DIR}/lj_ccall.c"     "${LUAJIT_SRC_DIR}/lj_ccallback.c" "${LUAJIT_SRC_DIR}/lj_carith.c"
    "${LUAJIT_SRC_DIR}/lj_clib.c"      "${LUAJIT_SRC_DIR}/lj_cparse.c"    "${LUAJIT_SRC_DIR}/lj_lib.c"
    "${LUAJIT_SRC_DIR}/lj_alloc.c"     "${LUAJIT_SRC_DIR}/lib_aux.c"       "${LUAJIT_SRC_DIR}/lib_init.c"
)

# 11. Final static library
add_library(LuaJIT STATIC ${LUAJIT_SOURCES} "${LUAJIT_VM_OUTPUT}")
add_dependencies(LuaJIT luaJIT_headers)

# Force LuaJIT to compile using strict C99 standards to preserve layout rules
set_target_properties(LuaJIT PROPERTIES
    C_STANDARD 99
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS ON
)

# Also enforce it on the host bootstrap utilities
set_target_properties(minilua PROPERTIES
    C_STANDARD 99
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS ON
)

set_target_properties(luaJIT_buildvm PROPERTIES
    C_STANDARD 99
    C_STANDARD_REQUIRED ON
    C_EXTENSIONS ON
)

if(UNIX AND NOT APPLE)
    target_compile_definitions(LuaJIT PRIVATE 
        _GNU_SOURCE 
        _FILE_OFFSET_BITS=64 
        _LARGEFILE_SOURCE
        LUAJIT_UNWIND_EXTERNAL   # <--- THIS IS THE KEY
    )
    
    # Linux specific libraries (Makefile Line 338)
    target_link_libraries(LuaJIT PRIVATE dl m)
endif()

target_include_directories(LuaJIT PUBLIC
    "${LUAJIT_SRC_DIR}"
    "${CMAKE_CURRENT_BINARY_DIR}"  # luajit.h, lj_bcdef.h, etc.
)
if(MSVC OR (WIN32 AND CMAKE_C_COMPILER_ID MATCHES "Clang"))
    target_compile_definitions(LuaJIT PRIVATE _CRT_SECURE_NO_DEPRECATE)
endif()
if(WIN32)
    target_compile_definitions(LuaJIT PRIVATE LUA_BUILD_AS_DLL)
endif()
if(NOT MSVC)
    target_compile_options(LuaJIT PRIVATE 
        -U_FORTIFY_SOURCE       # Makefile Line 217
        -fno-stack-protector    # Makefile Line 258 (Crucial for JIT stack)
        -fexceptions            # Required for external unwinding
        -fPIC                   # Standard for Linux shared/static interop
    )
endif()