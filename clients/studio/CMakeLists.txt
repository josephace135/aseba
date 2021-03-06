# kate: replace-tabs off; tab-width 4; indent-width 4; tab-indents true; indent-mode normal
# vim: ts=4:sw=4:noexpandtab
set(DOC_NAME aseba-doc)

# file(TO_NATIVE_PATH ...) is broken with MSYS generator
# (http://www.kwwidgets.org/Bug/bug_view_advanced_page.php?bug_id=5939)
# This is a replacement...
# From : http://www.cmake.org/pipermail/cmake/2006-July/010180.html
MACRO(MAKE_WINDOWS_PATH pathname)
	# An extra \\ escape is necessary to get a \ through CMake's processing.
	STRING(REPLACE "/" "\\" ${pathname} "${${pathname}}")
	# Enclose with UNESCAPED quotes.  This means we need to escape our
	# quotes once here, i.e. with \"
	SET(${pathname} \"${${pathname}}\")
ENDMACRO(MAKE_WINDOWS_PATH)

# link for Studio
set(ADDITIONAL_LIBRARIES "")

# Google protobuf
find_package(Protobuf)
if (PROTOBUF_FOUND AND PROTOBUF_PROTOC_EXECUTABLE)
	add_definitions(-DPROTOBUF_FOUND)
	include_directories(${PROTOBUF_INCLUDE_DIR})
	set(ADDITIONAL_LIBRARIES ${ADDITIONAL_LIBRARIES} ${PROTOBUF_LIBRARY})
	message(STATUS "Google Protobuf library and compiler found, enabling usage logging")
else ()
	message(STATUS "Google Protobuf library and compiler *not* found, usage logging disabled.")
endif ()

# Zeroconf
if (DNSSD_FOUND)
	set(ADDITIONAL_LIBRARIES ${ADDITIONAL_LIBRARIES} ${ASEBA_ZEROCONFQT_LIBRARIES})
endif()

if (Qt5Widgets_FOUND AND Qt5Xml_FOUND AND Qt5Network_FOUND AND Qt5Concurrent_FOUND AND Qt5Svg_FOUND)

	# add a console for debug messages under Windows
	if (WIN32)
		set(CMAKE_EXE_LINKER_FLAGS_DEBUG "-Wl,-subsystem,console ${CMAKE_EXE_LINKER_FLAGS}")
	endif ()

	if (QWT_LIBRARIES AND QWT_INCLUDE_DIR)
		message(STATUS "Qwt found, enabling plotting in Studio")
		include_directories (${QWT_INCLUDE_DIR})
		set(ADDITIONAL_LIBRARIES ${ADDITIONAL_LIBRARIES} ${QWT_LIBRARIES})
		add_definitions(-DHAVE_QWT)
	else ()
		message(STATUS "Qwt not found, disabling plotting in Studio")
	endif ()

	# generates the documentation
	if (Qt5Help_FOUND AND TARGET Qt5::qcollectiongenerator)
		message(STATUS "qcollectiongenerator executable found")
		get_property(QT_QCOLLECTIONGENERATOR_LOCATION TARGET Qt5::qcollectiongenerator PROPERTY IMPORTED_LOCATION)
		get_filename_component(QT_QCOLLECTIONGENERATOR_DIRECTORY ${QT_QCOLLECTIONGENERATOR_LOCATION} DIRECTORY)
		get_filename_component(QT_QCOLLECTIONGENERATOR_EXT ${QT_QCOLLECTIONGENERATOR_LOCATION} EXT)
		set(QT_QHELPGENERATOR_EXECUTABLE "${QT_QCOLLECTIONGENERATOR_DIRECTORY}/qhelpgenerator${QT_QCOLLECTIONGENERATOR_EXT}")
		set(QT_QCOLLECTIONGENERATOR_EXECUTABLE "${QT_QCOLLECTIONGENERATOR_DIRECTORY}/qcollectiongenerator${QT_QCOLLECTIONGENERATOR_EXT}")
		add_custom_command(
			OUTPUT ${DOC_NAME}.qch
			COMMAND ${QT_QHELPGENERATOR_EXECUTABLE} ARGS ${CMAKE_CURRENT_SOURCE_DIR}/${DOC_NAME}.qhp -o ${CMAKE_CURRENT_BINARY_DIR}/${DOC_NAME}.qch
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${DOC_NAME}.qhp)
		add_custom_command(
			OUTPUT ${DOC_NAME}.qhc
			COMMAND ${QT_QCOLLECTIONGENERATOR_EXECUTABLE} ARGS ${DOC_NAME}.qhcp -o ${DOC_NAME}.qhc
			DEPENDS ${DOC_NAME}.qhcp ${DOC_NAME}.qch)
	else ()
		message(WARNING "Qt help support missing, studio online help disabled (hint: on Ubuntu, install package qttools5-dev-tools)")
	endif ()

	# code common to the use of Aseba under Qt
	set(qt_common_SRCS
		Target.cpp
		DashelTarget.cpp
		TargetModels.cpp
		CustomWidgets.cpp
		AeslEditor.cpp
		Plugin.cpp
		translations/CompilerTranslator.cpp
	)
	qt5_wrap_cpp(qt_common_MOCS
		Target.h
		DashelTarget.h
		TargetModels.h
		AeslEditor.h
		translations/CompilerTranslator.h
	)
	add_library(asebaqtcommon STATIC ${qt_common_SRCS} ${qt_common_MOCS})
	qt5_use_modules(asebaqtcommon Gui Widgets Xml Network)

	# plugins
	add_subdirectory(plugins)

	# resources
	qt5_add_resources(resfiles asebastudio.qrc)

	include_directories (${CMAKE_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR})

	if (Qt5Help_FOUND)
		# studio for desktop use
		set(studio_SRCS
			StudioAeslEditor.cpp
			PluginRegistry.cpp
			MainWindow.cpp
			NamedValuesVectorModel.cpp
			ClickableLabel.cpp
			CustomDelegate.cpp
			main.cpp
			FindDialog.cpp
			EventViewer.cpp
			HelpViewer.cpp
			ConfigDialog.cpp
			ModelAggregator.cpp
		)
		qt5_wrap_cpp(studio_MOCS
			StudioAeslEditor.h
			MainWindow.h
			NamedValuesVectorModel.h
			ClickableLabel.h
			CustomDelegate.h
			FindDialog.h
			EventViewer.h
			HelpViewer.h
			ConfigDialog.h
			ModelAggregator.h
		)

		if (QT_QCOLLECTIONGENERATOR_EXECUTABLE)
			file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/asebastudiohelp.qrc ${CMAKE_CURRENT_SOURCE_DIR}/aseba-doc.qhcp DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
			qt5_add_resources(resfiles ${CMAKE_CURRENT_BINARY_DIR}/asebastudiohelp.qrc)
		endif ()

		add_executable(asebastudio WIN32 ${studio_SRCS} ${studio_MOCS} ${resfiles} ${CMAKE_BINARY_DIR}/version.h)
		if (HAS_DYN_VERSION)
			add_dependencies(asebastudio versionheader)
		endif ()

		add_dependencies(asebastudio asebaqtplugins asebaqtcommon asebaqtabout asebacompiler )
		target_link_libraries(asebastudio asebaqtplugins asebaqtcommon asebaqtabout asebacompiler ${ASEBA_CORE_LIBRARIES} ${ADDITIONAL_LIBRARIES} Qt5::Help Qt5::Svg Qt5::Concurrent Qt5::Network Qt5::Xml Qt5::Widgets)

		install(TARGETS asebastudio RUNTIME DESTINATION bin LIBRARY DESTINATION bin)
	endif ()

	# Thymio VPL standalone
	include_directories(${CMAKE_CURRENT_BINARY_DIR}/plugins/ThymioVPL)
	set(thymiovpl_SRCS
		ThymioVPLStandalone.cpp
		thymiovpl.cpp
	)
	qt5_wrap_cpp(thymiovpl_MOCS
		ThymioVPLStandalone.h
	)

	if (ANDROID)
		add_library(thymiovpl SHARED ${thymiovpl_SRCS} ${thymiovpl_MOCS} ${resfiles} ${CMAKE_BINARY_DIR}/version.h)
	else ()
		add_executable(thymiovpl WIN32 ${thymiovpl_SRCS} ${thymiovpl_MOCS} ${resfiles} ${CMAKE_BINARY_DIR}/version.h)
	endif ()
	if (HAS_DYN_VERSION)
		add_dependencies(thymiovpl versionheader)
	endif ()

	target_link_libraries(thymiovpl asebaqtplugins asebaqtcommon asebacompiler asebaqtabout ${ASEBA_CORE_LIBRARIES} ${ADDITIONAL_LIBRARIES} Qt5::Svg Qt5::Network Qt5::Xml Qt5::Widgets)

	# Render blocks helper
	set(rendervplblocks_SRCS
		rendervplblocks.cpp
	)
	add_executable(rendervplblocks WIN32 ${rendervplblocks_SRCS} ${resfiles} ${CMAKE_BINARY_DIR}/version.h)
	add_dependencies(rendervplblocks asebaqtplugins asebaqtcommon asebaqtabout asebacompiler)
	qt5_use_modules(rendervplblocks Xml)
	target_link_libraries(rendervplblocks asebaqtplugins asebaqtcommon asebaqtabout asebacompiler ${ASEBA_CORE_LIBRARIES} ${ADDITIONAL_LIBRARIES} Qt5::Svg Qt5::Widgets)

	install(TARGETS thymiovpl RUNTIME DESTINATION bin LIBRARY DESTINATION bin)
endif ()
