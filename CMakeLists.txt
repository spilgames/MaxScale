cmake_minimum_required(VERSION 2.6)

include(macros.cmake)

enable_testing()
set_variables() 
set_maxscale_version()

set(CMAKE_INSTALL_PREFIX "${INSTALL_DIR}" CACHE INTERNAL "Prefix prepended to install directories." FORCE)

project(MaxScale)

check_deps()
check_dirs()

set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_RPATH}:${CMAKE_INSTALL_PREFIX}/lib:${CMAKE_INSTALL_PREFIX}/modules)

configure_file(${CMAKE_SOURCE_DIR}/server/include/version.h.in ${CMAKE_SOURCE_DIR}/server/include/version.h)
configure_file(${CMAKE_SOURCE_DIR}/maxscale.conf.in ${CMAKE_SOURCE_DIR}/maxscale.conf.prep @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/etc/init.d/maxscale.in ${CMAKE_SOURCE_DIR}/etc/init.d/maxscale.prep @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/etc/ubuntu/init.d/maxscale.in ${CMAKE_SOURCE_DIR}/etc/ubuntu/init.d/maxscale.prep @ONLY)


set(CMAKE_C_FLAGS "-Wall -fPIC")
set(CMAKE_CXX_FLAGS "-Wall -fPIC")

if(BUILD_TYPE MATCHES Debug)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ggdb -pthread -pipe -DSS_DEBUG -Wformat -Werror=format-security -fstack-protector --param=ssp-buffer-size=4")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ggdb -pthread -pipe -DSS_DEBUG -Wformat -Werror=format-security -fstack-protector --param=ssp-buffer-size=4")
  message(STATUS "Generating debugging symbols")
elseif(BUILD_TYPE MATCHES Optimized)
  if(NOT (DEFINED OLEVEL))
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2")
	message(STATUS "Optimization level at: 2")
  endif()
else()

endif()

if(DEFINED OLEVEL )
  if((OLEVEL GREATER -1) AND (OLEVEL LESS 4) )
  	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O${OLEVEL}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O${OLEVEL}")
	message(STATUS "Optimization level at: ${OLEVEL}")
  else()
	message(WARNING "Optimization level was set to a bad value, ignoring it. (Valid values are 0-3)")
  endif()
endif()

if(GCOV)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lgcov")
endif()


subdirs(MYSQL_DIR_ALL ${MYSQL_DIR})
foreach(DIR ${MYSQL_DIR_ALL})
  include_directories(${DIR})
endforeach()

include_directories(${MYSQL_DIR}/..)
include_directories(utils)
include_directories(log_manager)
include_directories(query_classifier)
include_directories(server/include)
include_directories(server/inih)
include_directories(server/modules/include)

add_subdirectory(utils)
add_subdirectory(log_manager)
add_subdirectory(query_classifier)
add_subdirectory(server)
add_subdirectory(client)


# Install startup scripts and ldconfig files
if( NOT ( (DEFINED INSTALL_SYSTEM_FILES) AND ( NOT ( INSTALL_SYSTEM_FILES ) ) ) )

  install(FILES maxscale.conf.prep RENAME maxscale.conf DESTINATION /etc/ld.so.conf.d/ PERMISSIONS WORLD_EXECUTE WORLD_READ)
  if(DEB_BASED)
    install(FILES etc/ubuntu/init.d/maxscale.prep RENAME maxscale  DESTINATION /etc/init.d/ PERMISSIONS WORLD_EXECUTE)
  else()
    install(FILES etc/init.d/maxscale.prep RENAME maxscale DESTINATION /etc/init.d/ PERMISSIONS WORLD_EXECUTE)
  endif()
  message(STATUS "Installing maxscale.conf to: /etc/ld.so.conf.d")
  message(STATUS "Installing startup scripts to: /etc/init.d")
endif()

file(GLOB DOCS Documentation/*.pdf)
message(STATUS "Installing MaxScale to: ${CMAKE_INSTALL_PREFIX}/")

install(FILES server/MaxScale_template.cnf DESTINATION etc)
install(FILES ${ERRMSG} DESTINATION mysql)
install(FILES ${DOCS} DESTINATION Documentation)

# See if we are on a RPM-capable or DEB-capable system
find_program(RPMBUILD rpmbuild)
find_program(DEBBUILD dpkg-buildpackage)

if(NOT ( ${RPMBUILD} STREQUAL "RPMBUILD-NOTFOUND" ) )
  message(STATUS "Generating RPM packages")
  set(CPACK_GENERATOR "${CPACK_GENERATOR};RPM")
endif()

if(NOT ( ${DEBBUILD} STREQUAL "DEBBUILD-NOTFOUND" ) )
  set(CPACK_GENERATOR "${CPACK_GENERATOR};DEB")
  execute_process(COMMAND dpgk --print-architecture OUTPUT_VARIABLE DEB_ARCHITECTURE)
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${DEB_ARCHITECTURE})
  message(STATUS "Generating DEB packages for ${DEB_ARCHITECTURE}")
endif()

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "MaxScale")
set(CPACK_PACKAGE_VERSION_MAJOR "${MAXSCALE_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${MAXSCALE_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${MAXSCALE_VERSION_PATCH}")
set(CPACK_PACKAGE_CONTACT "SkySQL Ab")
set(CPACK_PACKAGE_FILE_NAME "maxscale-${MAXSCALE_VERSION}")
set(CPACK_PACKAGE_NAME "maxscale")
set(CPACK_PACKAGE_VENDOR "SkySQL Ab")
set(CPACK_PACKAGE_DESCRIPTION_FILE ${CMAKE_SOURCE_DIR}/README)
set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
set(CPACK_RPM_SPEC_INSTALL_POST "/sbin/ldconfig")
set(CPACK_RPM_PACKAGE_NAME "maxscale")
set(CPACK_RPM_PACKAGE_VENDOR "SkySQL Ab")
set(CPACK_RPM_PACKAGE_LICENSE "GPLv2")
set(CPACK_RPM_PACKAGE_AUTOREQPROV " no")
set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "/etc /etc/ld.so.conf.d /etc/init.d /etc/rc.d/init.d")
set(CPACK_RPM_SPEC_MORE_DEFINE "%define ignore \#")
set(CPACK_RPM_USER_FILELIST "%ignore /etc/init.d")
set(CPACK_RPM_USER_FILELIST "%ignore /etc/ld.so.conf.d")
set(CPACK_RPM_USER_FILELIST "%ignore /etc")
include(CPack)
add_custom_target(testall 
  COMMAND ${CMAKE_COMMAND} -DDEPS_OK=Y -DBUILD_TESTS=Y -DBUILD_TYPE=Debug -DINSTALL_DIR=${CMAKE_BINARY_DIR} -DINSTALL_SYSTEM_FILES=N ${CMAKE_SOURCE_DIR}
  COMMAND make install
  COMMAND /bin/sh -c "${CMAKE_BINARY_DIR}/bin/maxscale -c ${CMAKE_BINARY_DIR} &>/dev/null"
  COMMAND make test
  COMMAND /bin/sh -c "killall -KILL maxscale"
  COMMENT "Running full test suite")