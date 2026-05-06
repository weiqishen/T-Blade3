# ╔═══════════════════════════════════════════════════════════════════════════╗
# ║  T-Blade3 — CMake Build System                                         ║
# ║                                                                         ║
# ║  Turbomachinery blade parametric design tool.                           ║
# ║  Replaces the legacy Makefile/source/Makefile/MakefileESP build chain.  ║
# ║                                                                         ║
# ║  Quick start:                                                           ║
# ║    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release                       ║
# ║    cmake --build build                                                  ║
# ║                                                                         ║
# ║  Sections:                                                              ║
# ║    1. Project Setup & Options                                           ║
# ║    2. Output Directories                                                ║
# ║    3. Compiler Flags  (GCC / Intel)                                     ║
# ║    4. Source File Lists                                                  ║
# ║    5. Core Targets    (tblade3, techop, static library)                 ║
# ║    6. Shared Library  (optional, -DBUILD_SHARED_LIBS=ON)                ║
# ║    7. ESP UDP Plugins (optional, -DENABLE_ESP=ON)                       ║
# ║    8. pFUnit Tests    (optional, -DENABLE_TESTING=ON)                   ║
# ║    9. Install Rules                                                     ║
# ║   10. Configuration Summary                                            ║
# ╚═══════════════════════════════════════════════════════════════════════════╝

cmake_minimum_required(VERSION 3.20)
project(T-Blade3 LANGUAGES Fortran)


# ───────────────────────────────────────────────────────────────────────────
# 1.  Project Setup & Options
# ───────────────────────────────────────────────────────────────────────────

option(BUILD_SHARED_LIBS "Build shared library  (libTBlade3.so / .dylib)"    OFF)
option(ENABLE_ESP        "Build ESP/OpenCSM UDP plugins  (needs ESP_ROOT)"   OFF)
option(ENABLE_TESTING    "Build pFUnit test suite        (needs PFUNIT)"     OFF)
option(ENABLE_MEMCHECK   "Enable -fsanitize=address      (GCC only)"        OFF)
option(ENABLE_UNDEFINED  "Enable -fsanitize=undefined    (GCC only)"        OFF)


# ───────────────────────────────────────────────────────────────────────────
# 2.  Output Directories
#
#     Mirrors the original Makefile layout:
#       bin/   ← executables (tblade3, techop)
#       lib/   ← static & shared libraries
# ───────────────────────────────────────────────────────────────────────────

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/lib)

# Multi-config generators (Visual Studio, Xcode) append a per-config
# subdirectory by default.  Force all configs into the same flat layout.
foreach(_cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${_cfg} ${CMAKE_LIBRARY_OUTPUT_DIRECTORY})
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${_cfg} ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY})
endforeach()


# ───────────────────────────────────────────────────────────────────────────
# 3.  Compiler Flags
#
#     Build types follow standard CMake convention:
#       Debug          →  -O0, debug symbols, bounds checking
#       Release        →  -O2, no debug symbols
#       RelWithDebInfo →  -O2, debug symbols, traceback
# ───────────────────────────────────────────────────────────────────────────

if(CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")

    # -- Language standard ------------------------------------------------
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fdefault-real-8 -cpp")
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -ffree-form -ffree-line-length-none")
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fno-automatic")

    # -- Warnings ---------------------------------------------------------
    set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -Wall -Wno-unused-dummy-argument -Wline-truncation")

    # -- Build-type flags -------------------------------------------------
    set(CMAKE_Fortran_FLAGS_DEBUG          "-g -O0 -fbounds-check -fbacktrace")
    set(CMAKE_Fortran_FLAGS_RELEASE        "-O2")
    set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "-g -O2 -fbacktrace")

    # -- Position-independent code (needed for shared libs on Linux) ------
    if(NOT WIN32)
        set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fPIC")
    endif()

    # -- Optional sanitizers ----------------------------------------------
    if(ENABLE_MEMCHECK)
        set(CMAKE_Fortran_FLAGS    "${CMAKE_Fortran_FLAGS} -fsanitize=address -fno-omit-frame-pointer")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    endif()
    if(ENABLE_UNDEFINED)
        set(CMAKE_Fortran_FLAGS    "${CMAKE_Fortran_FLAGS} -fsanitize=undefined -fno-omit-frame-pointer")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=undefined")
    endif()

elseif(CMAKE_Fortran_COMPILER_ID STREQUAL "Intel" OR
       CMAKE_Fortran_COMPILER_ID STREQUAL "IntelLLVM")

    if(WIN32)
        # -- Windows Intel (ifort/ifx) — flags use '/' prefix -------------
        set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} /real-size:64 /fpp /free /warn:all")

        set(CMAKE_Fortran_FLAGS_DEBUG          "/debug:full /Od /check:bounds /traceback")
        set(CMAKE_Fortran_FLAGS_RELEASE        "/O2")
        set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "/O2 /debug:full /traceback")
    else()
        # -- Linux Intel (ifort/ifx) — flags use '-' prefix ---------------
        set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -real-size 64 -fpp -free -warn all -fPIC")

        set(CMAKE_Fortran_FLAGS_DEBUG          "-g -O0 -check bounds -traceback")
        set(CMAKE_Fortran_FLAGS_RELEASE        "-O2")
        set(CMAKE_Fortran_FLAGS_RELWITHDEBINFO "-g -O2 -traceback")
    endif()

else()
    message(WARNING
        "Untested Fortran compiler: ${CMAKE_Fortran_COMPILER_ID}. "
        "You may need to add the equivalent of -fdefault-real-8.")
endif()


# ───────────────────────────────────────────────────────────────────────────
# 4.  Source File Lists
#
#     Listed in topological (dependency) order.
#     CMake >=3.20 scans module deps automatically, but explicit ordering
#     keeps the intent clear and helps generators that don't scan.
# ───────────────────────────────────────────────────────────────────────────

set(SRC ${CMAKE_SOURCE_DIR}/source)

#  Core library — every module except program entry points
set(LIB_SOURCES
    ${SRC}/globvar.f90              # global variables
    ${SRC}/file_operations.f90      # I/O helpers
    ${SRC}/errors.f90               # error / warning routines
    ${SRC}/funcNsubs.f90            # math & geometry functions
    ${SRC}/derivatives.f90          # analytical derivatives
    ${SRC}/spline.f90               # spline evaluation
    ${SRC}/readinput.f90            # input-file parser
    ${SRC}/3dbgb.f90                # 3-D blade geometry builder
    ${SRC}/bladegen.f90             # blade section generator
    ${SRC}/bladestack.f90           # blade stacking
    ${SRC}/bspline3.f90             # cubic B-spline
    ${SRC}/cubicspline.f90          # cubic spline utilities
    ${SRC}/bsplinecam.f90           # camber-line B-spline
    ${SRC}/splinethick.f90          # thickness spline
    ${SRC}/airfoiltypes.f90         # predefined airfoil types
    ${SRC}/spanwise_variation.f90   # spanwise B-spline variation
)

#  techop — trailing-edge chopping tool (uses only a subset of modules)
set(TECHOP_SOURCES
    ${SRC}/globvar.f90
    ${SRC}/file_operations.f90
    ${SRC}/errors.f90
    ${SRC}/funcNsubs.f90
    ${SRC}/spline.f90
    ${SRC}/techop.f90
)


# ───────────────────────────────────────────────────────────────────────────
# 5.  Core Targets
#
#     Always built:
#       • TBlade3_static  — static library (lib/libTBlade3.a or .lib)
#       • tblade3          — main executable  (bin/tblade3)
#       • techop           — TE-chop tool     (bin/techop)
# ───────────────────────────────────────────────────────────────────────────

# Place all .mod files in one directory, out of the source tree
set(MOD_DIR ${CMAKE_BINARY_DIR}/mod)

# --- static library (also consumed by ESP and test targets) ---------------
add_library(TBlade3_static STATIC ${LIB_SOURCES})
set_target_properties(TBlade3_static PROPERTIES
    OUTPUT_NAME              "TBlade3"
    Fortran_MODULE_DIRECTORY ${MOD_DIR}
)
target_include_directories(TBlade3_static PUBLIC ${MOD_DIR})

# --- tblade3 executable ---------------------------------------------------
add_executable(tblade3 ${SRC}/3dbgb_driver.f90)
target_link_libraries(tblade3 PRIVATE TBlade3_static)
target_include_directories(tblade3 PRIVATE ${MOD_DIR})

# --- techop executable ----------------------------------------------------
add_executable(techop ${TECHOP_SOURCES})
set_target_properties(techop PROPERTIES Fortran_MODULE_DIRECTORY ${MOD_DIR})


# ───────────────────────────────────────────────────────────────────────────
# 6.  Shared Library  (cmake -DBUILD_SHARED_LIBS=ON)
#
#     Produces libTBlade3.so (Linux) or libTBlade3.dylib (macOS).
#     Equivalent to the original `make library` target.
# ───────────────────────────────────────────────────────────────────────────

if(BUILD_SHARED_LIBS)
    add_library(TBlade3_shared SHARED ${LIB_SOURCES})
    set_target_properties(TBlade3_shared PROPERTIES
        OUTPUT_NAME              "TBlade3"
        Fortran_MODULE_DIRECTORY ${MOD_DIR}
    )
    install(TARGETS TBlade3_shared LIBRARY DESTINATION lib)
endif()


# ───────────────────────────────────────────────────────────────────────────
# 7.  ESP / OpenCSM UDP Plugins  (cmake -DENABLE_ESP=ON -DESP_ROOT=...)
#
#     Equivalent to the original `make esp` / MakefileESP.
#
#     Builds:
#       lib/libTBlade3.a     — Fortran static lib compiled with -DFROM_ESP
#       lib/tblade.so        — UDP plugin → udpTblade.c
#       lib/bladevolume.so   — UDP plugin → udpBladeVolume.c
#       lib/hubwedge.so      — UDP plugin → udpHubWedge.c
#
#     All plugins link against libTBlade3 and libegads from ESP_ROOT.
# ───────────────────────────────────────────────────────────────────────────

if(ENABLE_ESP)
    enable_language(C)

    # -- Validate ESP_ROOT ------------------------------------------------
    if(NOT DEFINED ESP_ROOT)
        message(FATAL_ERROR
            "ENABLE_ESP=ON but ESP_ROOT is not set.\n"
            "  cmake -DENABLE_ESP=ON -DESP_ROOT=/path/to/ESP ..")
    endif()
    set(ESP_INC ${ESP_ROOT}/include)
    set(ESP_LIB ${ESP_ROOT}/lib)
    if(NOT EXISTS ${ESP_INC}/udpUtilities.h)
        message(FATAL_ERROR "Cannot find ${ESP_INC}/udpUtilities.h — check ESP_ROOT.")
    endif()

    # -- Fortran library rebuilt with FROM_ESP -----------------------------
    add_library(TBlade3_esp STATIC ${LIB_SOURCES} ${SRC}/3dbgb_driver.f90)
    target_compile_definitions(TBlade3_esp PRIVATE FROM_ESP)
    set_target_properties(TBlade3_esp PROPERTIES
        OUTPUT_NAME              "TBlade3"
        Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/mod_esp
    )

    # -- Helper: add one UDP shared-library plugin ------------------------
    #    _add_udp_plugin(<target>  <output_name>  <c_source>)
    function(_add_udp_plugin _target _name _csrc)
        add_library(${_target} SHARED ${_csrc})
        target_include_directories(${_target} PRIVATE ${ESP_INC})
        target_link_libraries(${_target} PRIVATE TBlade3_esp)
        target_link_directories(${_target} PRIVATE ${ESP_LIB})
        target_link_libraries(${_target} PRIVATE egads)
        set_target_properties(${_target} PROPERTIES OUTPUT_NAME ${_name} PREFIX "")
    endfunction()

    _add_udp_plugin(tblade_udp      tblade      ${SRC}/udpTblade.c)
    _add_udp_plugin(bladevolume_udp bladevolume  ${SRC}/udpBladeVolume.c)
    _add_udp_plugin(hubwedge_udp    hubwedge    ${SRC}/udpHubWedge.c)

    install(TARGETS TBlade3_esp                              ARCHIVE DESTINATION lib)
    install(TARGETS tblade_udp bladevolume_udp hubwedge_udp   LIBRARY DESTINATION lib)
endif()


# ───────────────────────────────────────────────────────────────────────────
# 8.  pFUnit Tests  (cmake -DENABLE_TESTING=ON -DPFUNIT=/path/to/pFUnit)
#
#     Equivalent to the original `make tests` target.
#
#     Pipeline:  .pf ──pFUnitParser──▶ .F90 ──compile──▶ tests executable
#     The test binary is placed in tests/ and registered with CTest.
# ───────────────────────────────────────────────────────────────────────────

if(ENABLE_TESTING)
    enable_testing()

    # -- Validate PFUNIT path ---------------------------------------------
    if(NOT DEFINED PFUNIT)
        message(FATAL_ERROR
            "ENABLE_TESTING=ON but PFUNIT is not set.\n"
            "  cmake -DENABLE_TESTING=ON -DPFUNIT=/path/to/pFUnit ..")
    endif()
    if(NOT EXISTS ${PFUNIT}/include/driver.F90)
        message(FATAL_ERROR "Cannot find ${PFUNIT}/include/driver.F90 — check PFUNIT.")
    endif()

    set(TEST_DIR ${CMAKE_SOURCE_DIR}/tests)

    # -- Preprocess .pf → .F90 via pFUnitParser ----------------------------
    file(GLOB PF_SRCS ${TEST_DIR}/*.pf)
    set(GEN_SRCS "")

    foreach(_pf ${PF_SRCS})
        get_filename_component(_name ${_pf} NAME_WE)
        set(_gen ${CMAKE_BINARY_DIR}/tests/${_name}.F90)
        add_custom_command(
            OUTPUT  ${_gen}
            COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/tests
            COMMAND python ${PFUNIT}/bin/pFUnitParser.py ${_pf} ${_gen}
            DEPENDS ${_pf}
            COMMENT "pFUnit: ${_name}.pf → ${_name}.F90"
        )
        list(APPEND GEN_SRCS ${_gen})
    endforeach()

    # -- Build the test executable -----------------------------------------
    add_executable(tests_x ${PFUNIT}/include/driver.F90 ${GEN_SRCS})
    target_link_libraries(tests_x PRIVATE TBlade3_static)
    target_include_directories(tests_x PRIVATE
        ${PFUNIT}/mod  ${PFUNIT}/include  ${TEST_DIR}  ${MOD_DIR}
    )
    target_link_directories(tests_x PRIVATE ${PFUNIT}/lib)
    target_link_libraries(tests_x PRIVATE pfunit)
    target_compile_definitions(tests_x PRIVATE GNU BUILD_ROBUST)
    set_target_properties(tests_x PROPERTIES
        OUTPUT_NAME              "tests"
        Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/tests
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/tests
    )

    # -- Register with CTest -----------------------------------------------
    add_test(NAME T-Blade3_pFUnit
             COMMAND tests_x -xml ${CMAKE_SOURCE_DIR}/tests/tests.xml
             WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

    message(STATUS "pFUnit tests found: ${PF_SRCS}")
endif()


# ───────────────────────────────────────────────────────────────────────────
# 9.  Install Rules    (cmake --install build --prefix /usr/local)
# ───────────────────────────────────────────────────────────────────────────

install(TARGETS tblade3 techop   RUNTIME DESTINATION bin)
install(TARGETS TBlade3_static   ARCHIVE DESTINATION lib)
install(DIRECTORY ${MOD_DIR}/    DESTINATION include/TBlade3
        FILES_MATCHING PATTERN "*.mod")


# ───────────────────────────────────────────────────────────────────────────
# 10. Configuration Summary
# ───────────────────────────────────────────────────────────────────────────

message(STATUS "")
message(STATUS "╔═══════════════════════════════════════╗")
message(STATUS "║  T-Blade3 Build Configuration         ║")
message(STATUS "╠═══════════════════════════════════════╣")
message(STATUS "║  Compiler:       ${CMAKE_Fortran_COMPILER_ID} ${CMAKE_Fortran_COMPILER_VERSION}")
message(STATUS "║  Build type:     ${CMAKE_BUILD_TYPE}")
message(STATUS "║  Shared library: ${BUILD_SHARED_LIBS}")
message(STATUS "║  ESP plugins:    ${ENABLE_ESP}")
message(STATUS "║  Testing:        ${ENABLE_TESTING}")
message(STATUS "║  Memcheck:       ${ENABLE_MEMCHECK}")
message(STATUS "║  Undefined:      ${ENABLE_UNDEFINED}")
message(STATUS "║  Install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "╚═══════════════════════════════════════╝")
message(STATUS "")
