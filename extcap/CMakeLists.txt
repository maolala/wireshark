# CMakeLists.txt
#
# Wireshark - Network traffic analyzer
# By Gerald Combs <gerald@wireshark.org>
# Copyright 1998 Gerald Combs
#
# SPDX-License-Identifier: GPL-2.0-or-later
#

if(EXTCAP_ANDROIDDUMP_LIBPCAP)
	set(ANDROIDDUMP_USE_LIBPCAP 1)
endif()

if(LIBSSH_FOUND)
	SET(CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES} ${LIBSSH_LIBRARIES})
	CHECK_FUNCTION_EXISTS(ssh_userauth_agent LIBSSH_USERAUTH_AGENT_FOUND)
	if(LIBSSH_USERAUTH_AGENT_FOUND)
		set(HAVE_SSH_USERAUTH_AGENT 1)
	endif()
endif()

# Ensure "run/extcap" exists
# add_custom_command(OUTPUT "${DATAFILE_DIR}/extcap"
# 	COMMAND ${CMAKE_COMMAND} -E make_directory
# 		"${DATAFILE_DIR}/extcap"
# )
# list(APPEND copy_data_files_depends "${DATAFILE_DIR}/extcap")


macro(set_extcap_executable_properties _executable)
	set_target_properties(${_executable} PROPERTIES FOLDER "Executables/Extcaps")

	set(PROGLIST ${PROGLIST} ${_executable})

	if(WIN32)
		set_target_properties(${_executable} PROPERTIES
			LINK_FLAGS "${WS_LINK_FLAGS}"
			RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/run/extcap
			RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR}/run/Debug/extcap
			RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR}/run/Release/extcap
			RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_BINARY_DIR}/run/MinSizeRel/extcap
			RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_BINARY_DIR}/run/RelWithDebInfo/extcap
		)
	else()
		set_target_properties(${_executable} PROPERTIES
			LINK_FLAGS "${WS_LINK_FLAGS}"
			RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/run/extcap
		)
		if(ENABLE_RPATH_ORIGIN)
			set_target_properties(${_executable} PROPERTIES
				INSTALL_RPATH "\$ORIGIN/../.."
			)
		endif()
		if(ENABLE_APPLICATION_BUNDLE)
			if(NOT CMAKE_CFG_INTDIR STREQUAL ".")
				# Xcode
				set_target_properties(${_executable} PROPERTIES
					RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/run/$<CONFIG>/Wireshark.app/Contents/MacOS/extcap
				)
			else()
				set_target_properties(${_executable} PROPERTIES
					RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/run/Wireshark.app/Contents/MacOS/extcap
				)
				# Add a wrapper script which runs each executable from the
				# correct location. This adds convenience but makes debugging
				# more difficult.
				file(REMOVE ${CMAKE_BINARY_DIR}/run/${_executable})
				file(WRITE ${CMAKE_BINARY_DIR}/run/${_executable} "#!/bin/sh\n")
				file(APPEND ${CMAKE_BINARY_DIR}/run/${_executable} "exec ${CMAKE_BINARY_DIR}/run/Wireshark.app/Contents/MacOS/extcap/${_executable} \"\$\@\"\n")
				execute_process(COMMAND chmod a+x ${CMAKE_BINARY_DIR}/run/${_executable})
			endif()
		endif()
	endif()
endmacro()

add_custom_target(extcaps)

if(BUILD_androiddump)
	if(EXTCAP_ANDROIDDUMP_LIBPCAP)
		if(HAVE_LIBPCAP)
			set(androiddump_LIBS
				ui
				${GLIB2_LIBRARIES}
				${PCAP_LIBRARIES}
			)
		else()
			message(FATAL_ERROR "You are trying to build androiddump with libpcap but do not have it")
		endif()
	else()
		set(androiddump_LIBS
			ui
			wiretap
			${GLIB2_LIBRARIES}
			${ZLIB_LIBRARIES}
			${CMAKE_DL_LIBS}
		)
	endif()
	set(androiddump_FILES
		../cli_main.c
		androiddump.c
		extcap-base.c
	)

	set_executable_resources(androiddump "Androiddump")
	add_executable(androiddump ${androiddump_FILES})
	# XXX Shouldn't we add wsutil to androiddump_LIBS instead?
	set_extcap_executable_properties(androiddump)
	target_link_libraries(androiddump ${androiddump_LIBS})
	install(TARGETS androiddump RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps androiddump)
endif()

if(BUILD_sshdump AND LIBSSH_FOUND)
	set(sshdump_LIBS
		wsutil
		${GLIB2_LIBRARIES}
		${CMAKE_DL_LIBS}
		${LIBSSH_LIBRARIES}
	)
	set(sshdump_FILES
		../cli_main.c
		sshdump.c
		extcap-base.c
		ssh-base.c
	)

	set_executable_resources(sshdump "Sshdump")
	add_executable(sshdump ${sshdump_FILES})
	set_extcap_executable_properties(sshdump)
	target_link_libraries(sshdump ${sshdump_LIBS})
	target_include_directories(sshdump PUBLIC ${LIBSSH_INCLUDE_DIR})
	install(TARGETS sshdump RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps sshdump)
elseif (BUILD_sshdump)
	#message( WARNING "Cannot find libssh, cannot build sshdump" )
endif()

if(BUILD_ciscodump AND LIBSSH_FOUND)
	set(ciscodump_LIBS
		writecap
		wsutil
		${GLIB2_LIBRARIES}
		${CMAKE_DL_LIBS}
		${LIBSSH_LIBRARIES}
	)
	set(ciscodump_FILES
		../cli_main.c
		ciscodump.c
		extcap-base.c
		ssh-base.c
	)

	set_executable_resources(ciscodump "Ciscodump")
	add_executable(ciscodump ${ciscodump_FILES})
	set_extcap_executable_properties(ciscodump)
	target_link_libraries(ciscodump ${ciscodump_LIBS})
	target_include_directories(ciscodump PUBLIC ${LIBSSH_INCLUDE_DIR})
	install(TARGETS ciscodump RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps ciscodump)
elseif (BUILD_ciscodump)
	#message( WARNING "Cannot find libssh, cannot build ciscodump" )
endif()

if(BUILD_dpauxmon AND HAVE_LIBNL3)
	set(dpauxmon_LIBS
		${GLIB2_LIBRARIES}
		${CMAKE_DL_LIBS}
		wsutil
		writecap
		${NL_LIBRARIES}
	)
	set(dpauxmon_FILES
		dpauxmon.c
		extcap-base.c
	)

	set_executable_resources(dpauxmon "dpauxmon")
	add_executable(dpauxmon ${dpauxmon_FILES})
	set_extcap_executable_properties(dpauxmon)
	target_link_libraries(dpauxmon ${dpauxmon_LIBS})
	target_include_directories(dpauxmon PUBLIC ${NL_INCLUDE_DIR})
	install(TARGETS dpauxmon RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps dpauxmon)
elseif (BUILD_dpauxmon)
	#message( WARNING "Cannot find libnl3, cannot build dpauxmon" )
endif()

if(BUILD_udpdump)
	set(udpdump_LIBS
		${GLIB2_LIBRARIES}
		${CMAKE_DL_LIBS}
		wsutil
		writecap
	)
	set(udpdump_FILES
		../cli_main.c
		udpdump.c
		extcap-base.c
	)

	set_executable_resources(udpdump "udpdump")
	add_executable(udpdump ${udpdump_FILES})
	set_extcap_executable_properties(udpdump)
	target_link_libraries(udpdump ${udpdump_LIBS})
	install(TARGETS udpdump RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps udpdump)
endif()

if(BUILD_randpktdump)
	set(randpktdump_LIBS
		randpkt_core
		ui
		wiretap
		${GLIB2_LIBRARIES}
		${ZLIB_LIBRARIES}
		${CMAKE_DL_LIBS}
	)
	set(randpktdump_FILES
		../cli_main.c
		extcap-base.c
		randpktdump.c
	)

	set_executable_resources(randpktdump "randpktdump")
	add_executable(randpktdump ${randpktdump_FILES})
	# XXX Shouldn't we add wsutil to randpktdump_LIBS instead?
	set_extcap_executable_properties(randpktdump)
	target_link_libraries(randpktdump ${randpktdump_LIBS})
	install(TARGETS randpktdump RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps randpktdump)
endif()

if(BUILD_sdjournal AND SYSTEMD_FOUND)
	set(sdjournal_LIBS
		ui
		wiretap
		writecap
		wsutil
		${GLIB2_LIBRARIES}
		${ZLIB_LIBRARIES}
		${CMAKE_DL_LIBS}
		${SYSTEMD_LIBRARIES}
	)
	set(sdjournal_FILES
		extcap-base.c
		sdjournal.c
	)

	set_executable_resources(sdjournal "sdjournal")
	add_executable(sdjournal ${sdjournal_FILES})
	set_extcap_executable_properties(sdjournal)
	target_link_libraries(sdjournal ${sdjournal_LIBS})
	install(TARGETS sdjournal RUNTIME DESTINATION ${EXTCAP_INSTALL_LIBDIR})
	add_dependencies(extcaps sdjournal)
elseif (BUILD_sdjournal)
	#message( WARNING "Cannot find libsystemd, cannot build sdjournal" )
endif()

set(CLEAN_C_FILES
	${dumpcap_FILES}
	${androiddump_FILES}
	${sshdump_FILES}
	${ciscodump_FILES}
)
