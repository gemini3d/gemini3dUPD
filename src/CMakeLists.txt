add_compile_options(${gemini3d_flags})
# scope is this directory on down

# --- just use one function from LAPACK95
add_subdirectory(vendor/lapack95)  # used by PDEparabolic

# --- GEMINI
add_subdirectory(ionization)

add_subdirectory(numerical)
# we use this even with MPI stub
add_subdirectory(mpimod)

add_subdirectory(io)
add_subdirectory(temporal)

add_library(collision OBJECT collisions/collisions.f90)
target_link_libraries(collision PRIVATE const gemini3d_config meshobj)

# --- other libs

add_subdirectory(neutral)
add_subdirectory(sources)

add_library(multifluid_mpi OBJECT multifluid/multifluid_mpi.f90)
target_link_libraries(multifluid_mpi PRIVATE const mpimod advec_mpi)

add_library(multifluid OBJECT multifluid/multifluid.f90
$<TARGET_OBJECTS:grid>
)
target_link_libraries(multifluid PRIVATE
advec calculus collision const diffusion meshobj precipdata ionization precipBCs sources timeutils gemini3d_config
)

add_library(gemini_work_def OBJECT intvars/gemini_work_def.f90)
target_link_libraries(gemini_work_def PRIVATE const inputdata neutral precipdata efielddata
	neutraldata neutraldata3D neutraldata3D_fclaw)

add_subdirectory(utils)
add_subdirectory(inputdata)

# --- Main GEMINI3D library

configure_file(gemini_cli.in.f90 gemini_cli.f90 @ONLY)

add_library(gemini3d libgemini.f90 gemini_init.f90
${CMAKE_CURRENT_BINARY_DIR}/gemini_cli.f90
$<TARGET_OBJECTS:exe_frontend>
)
target_include_directories(gemini3d PUBLIC
"$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include;${MPI_Fortran_INCLUDE_DIRS}>"
$<INSTALL_INTERFACE:include>
)
target_link_libraries(gemini3d PUBLIC
h5fortran::h5fortran
${MPI_Fortran_LIBRARIES}
$<$<BOOL:${glow}>:glow::glow>
$<$<BOOL:${hwm14}>:hwm14::hwm_ifc>
msis::msis
ffilesystem::filesystem
)
# need to have this explicit include for use of Gemini3D from ExternalProject
# else compiler will fail to include top-level build/include for some targets e.g. filesystem

add_library(gemini3d_mpi libgemini_mpi.f90)

set_target_properties(gemini3d gemini3d_mpi PROPERTIES
ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
)

set(gemini_internal_mods
advec
calculus gemini3d_config
const
collision
diffusion fang interp
ionization
inputdata precipdata efielddata solfluxdata
gemini3d_sysinfo
neutral
neutraldata neutraldata3D neutraldata3D_fclaw neutraldata3D_fclaw_axisymm neutraldata3D_fclaw_3Dx
neutraldata3D_mpi neutraldata3D_geom_mpi neutraldata3D_geog_mpi
neutraldata2D neutraldata2Daxisymm neutraldata2Dcart
meshobj meshobj_cart meshobj_dipole spherical newton geomagnetic dipole
multifluid
PDEparabolic gbsv
precipBCs solfluxBCs reader sanity_check errors sources temporal timeutils
exe_frontend
grid io
mpimod
autogrid
io_nompi
potential_nompi
)

set(gemini_mpi_internal_mods
advec
advec_mpi
calculus gemini3d_config
const
collision
diffusion fang interp
ionization
ionization_mpi
inputdata precipdata efielddata solfluxdata
gemini3d_sysinfo
neutral
neutraldata neutraldata3D neutraldata3D_fclaw neutraldata3D_fclaw_axisymm neutraldata3D_fclaw_3Dx
neutraldata3D_mpi neutraldata3D_geom_mpi neutraldata3D_geog_mpi
neutraldata2D neutraldata2Daxisymm neutraldata2Dcart
meshobj meshobj_cart meshobj_dipole spherical newton geomagnetic dipole
multifluid
multifluid_mpi
neutral_perturbations
PDEparabolic gbsv
precipBCs solfluxBCs reader sanity_check errors sources sources_mpi temporal_mpi timeutils
exe_frontend
grid io
PDEelliptic potential mumps_interface
mpimod
autogrid
)

# --- Gemini.bin minimal main Fortran program
add_executable(gemini.bin ${PROJECT_SOURCE_DIR}/app/main.f90)
set_target_properties(gemini.bin PROPERTIES
RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
RELWITHDEBINFO_POSTFIX .debug
DEBUG_POSTFIX .debug
)
target_link_libraries(gemini.bin PRIVATE
gemini3d_mpi
gemini3d
h5fortran::h5fortran
MUMPS::MUMPS
$<$<BOOL:${glow}>:glow::glow>
$<$<BOOL:${hwm14}>:hwm14::hwm_ifc>
ffilesystem::filesystem
)
if(CMAKE_Fortran_COMPILER_ID MATCHES "^Intel")
  set_target_properties(gemini.bin PROPERTIES LINKER_LANGUAGE Fortran)
else()
  set_target_properties(gemini.bin PROPERTIES LINKER_LANGUAGE CXX)
endif()

# --- Gemini.bin main Fortran program for solving density and potential only
add_executable(gemini.denspot.bin ${PROJECT_SOURCE_DIR}/app/main.denspot.f90)
set_target_properties(gemini.denspot.bin PROPERTIES
RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
RELWITHDEBINFO_POSTFIX .debug
DEBUG_POSTFIX .debug
)
target_link_libraries(gemini.denspot.bin PRIVATE
gemini3d_mpi
gemini3d
h5fortran::h5fortran
MUMPS::MUMPS
$<$<BOOL:${glow}>:glow::glow>
$<$<BOOL:${hwm14}>:hwm14::hwm_ifc>
ffilesystem::filesystem
)
if(CMAKE_Fortran_COMPILER_ID MATCHES "^Intel")
  set_target_properties(gemini.denspot.bin PROPERTIES LINKER_LANGUAGE Fortran)
else()
  set_target_properties(gemini.denspot.bin PROPERTIES LINKER_LANGUAGE CXX)
endif()

# --- Gemini.bin main Fortran program with alternative energy solution
add_executable(gemini.altenergy.bin ${PROJECT_SOURCE_DIR}/app/main.altenergy.f90)
set_target_properties(gemini.denspot.bin PROPERTIES
RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
RELWITHDEBINFO_POSTFIX .debug
DEBUG_POSTFIX .debug
)
target_link_libraries(gemini.altenergy.bin PRIVATE
gemini3d_mpi
gemini3d
h5fortran::h5fortran
MUMPS::MUMPS
$<$<BOOL:${glow}>:glow::glow>
$<$<BOOL:${hwm14}>:hwm14::hwm_ifc>
ffilesystem::filesystem
)
if(CMAKE_Fortran_COMPILER_ID MATCHES "^Intel")
  set_target_properties(gemini.altenergy.bin PROPERTIES LINKER_LANGUAGE Fortran)
else()
  set_target_properties(gemini.altenergy.bin PROPERTIES LINKER_LANGUAGE CXX)
endif()

if(hwm14)
  # HWM14 data files
  # due to legacy coding style, HWM14 files need to be in binary directory
  set(hwm14_data_dir "${hwm14_SOURCE_DIR}/data/")
  if(NOT IS_DIRECTORY ${hwm14_data_dir})
    message(FATAL_ERROR "HWM14 data directory not found: ${hwm14_data_dir}")
  endif()

  set(hwm14_RESOURCE_FILES
  ${hwm14_data_dir}/hwm123114.bin
  ${hwm14_data_dir}/dwm07b104i.dat
  ${hwm14_data_dir}/gd2qd.dat
  )

  # this is to run from the live build directory
  add_custom_command(TARGET gemini.bin POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${hwm14_RESOURCE_FILES} $<TARGET_FILE_DIR:gemini.bin>
  COMMAND_EXPAND_LISTS
  )

  # this is for installed Gemini3D, which in general may be installed
  # somewhere other than HWM14 directory tree
  install(FILES ${hwm14_RESOURCE_FILES} TYPE BIN)

endif(hwm14)

# same link logic applied to test/test_mumps

# need this header for external projects using Gemini3D
install(FILES ${PROJECT_SOURCE_DIR}/include/gemini3d.h TYPE INCLUDE)

add_library(gemini3d_c libgemini_c.f90)
target_link_libraries(gemini3d_c PRIVATE gemini3d)

add_library(gemini3d_mpi_c libgemini_mpi_c.f90)
target_link_libraries(gemini3d_mpi_c PRIVATE gemini3d_mpi gemini3d_c)

foreach(l IN LISTS gemini_internal_mods)
  foreach(t IN ITEMS gemini3d gemini.bin gemini3d_c)
    target_sources(${t} PRIVATE $<TARGET_OBJECTS:${l}>)
  endforeach()
endforeach()

target_link_libraries(gemini3d_mpi PRIVATE gemini3d)

foreach(l IN LISTS gemini_mpi_internal_mods)
  target_sources(gemini3d_mpi PRIVATE $<TARGET_OBJECTS:${l}>)
endforeach()

install(TARGETS gemini.bin gemini3d gemini3d_mpi gemini3d_mpi_c gemini3d_c
EXPORT ${PROJECT_NAME}-targets
)

# --- Gemini_c.bin
# C++ main program just like Fortran gemini.bin

add_library(gemini3d_cpp libgemini.cpp)
target_include_directories(gemini3d_cpp PUBLIC
$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
$<INSTALL_INTERFACE:include>
)
target_link_libraries(gemini3d_cpp PUBLIC
gemini3d_mpi_c
gemini3d_mpi
gemini3d_c
h5fortran::h5fortran
MUMPS::MUMPS
MPI::MPI_CXX
$<$<BOOL:${glow}>:glow::glow>
ffilesystem::filesystem
)
target_compile_definitions(gemini3d_cpp PRIVATE
$<$<BOOL:${MSVC}>:_CRT_SECURE_NO_WARNINGS>
)
#$<$<BOOL:${MSVC}>:_ALLOW_ITERATOR_DEBUG_LEVEL_MISMATCH>
#$<$<BOOL:${MSVC}>:_HAS_ITERATOR_DEBUGGING=1>
#$<$<BOOL:${MSVC}>:_ITERATOR_DEBUG_LEVEL=2>
install(TARGETS gemini3d_cpp EXPORT ${PROJECT_NAME}-targets)

# C++ Gemini3D executable
if(HAVE_CXX_FILESYSTEM)

add_executable(gemini_c.bin ${PROJECT_SOURCE_DIR}/app/main.cpp)
set_target_properties(gemini_c.bin PROPERTIES
RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
#LINKER_LANGUAGE CXX
RELWITHDEBINFO_POSTFIX .debug
DEBUG_POSTFIX .debug
)
target_include_directories(gemini_c.bin PUBLIC
$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
$<INSTALL_INTERFACE:include>
)
target_link_libraries(gemini_c.bin PRIVATE
gemini3d_cpp
gemini3d_mpi_c
gemini3d_mpi
gemini3d_c
gemini3d
)
target_compile_definitions(gemini_c.bin PRIVATE $<$<BOOL:${MSVC}>:_CRT_SECURE_NO_WARNINGS>)
target_compile_features(gemini_c.bin PRIVATE cxx_std_17)

install(TARGETS gemini_c.bin EXPORT ${PROJECT_NAME}-targets)

endif()
