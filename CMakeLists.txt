# Use FetchContent to avoid directly distributing huge MUMPS source,
# which would slow down clone of this repo

cmake_minimum_required(VERSION 3.13...3.20)

project(MUMPS
LANGUAGES C Fortran
VERSION 5.4.0
DESCRIPTION "Sparse direct parallel solver"
HOMEPAGE_URL "http://mumps-solver.org/")

include(CTest)
include(FetchContent)

if(NOT DEFINED ${PROJECT_NAME}_BUILD_TESTING)
  set(${PROJECT_NAME}_BUILD_TESTING ${BUILD_TESTING})
endif()

# --- user options
include(cmake/options.cmake)
include(cmake/libraries.cmake)

# --- compiler setup
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules)

include(cmake/compilers.cmake)

# --- download MUMPS
include(cmake/get_mumps.cmake)

# --- dependencies
find_package(Threads)

include(cmake/lapack.cmake)

# -- Scalapack / MPI

if(parallel)
  include(cmake/mpi.cmake)
  include(cmake/scalapack.cmake)

  set(NUMERIC_LIBS SCALAPACK::SCALAPACK LAPACK::LAPACK MPI::MPI_Fortran MPI::MPI_C)
  if(openmp)
    find_package(OpenMP COMPONENTS C Fortran REQUIRED)
    list(APPEND NUMERIC_LIBS OpenMP::OpenMP_Fortran OpenMP::OpenMP_C)
  endif()
else(parallel)
  configure_file(libseq/CMakeLists.txt ${mumps_SOURCE_DIR}/libseq/ COPYONLY)
  add_subdirectory(${mumps_SOURCE_DIR}/libseq ${mumps_BINARY_DIR}/libseq)
  set(NUMERIC_LIBS mpiseq)
endif(parallel)
list(APPEND NUMERIC_LIBS LAPACK::LAPACK ${CMAKE_THREAD_LIBS_INIT})

# --- ordering libs

set(ORDERING_LIBS)
set(ORDERING_FLAGS -Dpord)
if(scotch)
  find_package(Scotch COMPONENTS ESMUMPS REQUIRED)
  list(APPEND ORDERING_FLAGS -Dscotch)
  list(APPEND ORDERING_LIBS Scotch::Scotch)
endif()
if(metis)
  find_package(METIS REQUIRED)
  list(APPEND ORDERING_FLAGS -Dmetis)
  list(APPEND ORDERING_LIBS METIS::METIS)
endif()
list(APPEND ORDERING_LIBS pord)

install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindLAPACK.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindSCALAPACK.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindBLACS.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindMETIS.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindScotch.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/FindMUMPS.cmake
  DESTINATION lib/cmake/${PROJECT_NAME}/Modules)


# --- MUMPS build

configure_file(PORD/CMakeLists.txt ${mumps_SOURCE_DIR}/PORD/ COPYONLY)
configure_file(PORD/lib/CMakeLists.txt ${mumps_SOURCE_DIR}/PORD/lib/ COPYONLY)
add_subdirectory(${mumps_SOURCE_DIR}/PORD ${mumps_BINARY_DIR}/PORD)

configure_file(src/CMakeLists.txt ${mumps_SOURCE_DIR}/src/ COPYONLY)
add_subdirectory(${mumps_SOURCE_DIR}/src ${mumps_BINARY_DIR}/src)


if(${PROJECT_NAME}_BUILD_TESTING)
  add_subdirectory(tests)
endif()


# --- feature summary

include(cmake/summary.cmake)

include(cmake/install.cmake)