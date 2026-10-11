FIND_PACKAGE(SWIG REQUIRED)
INCLUDE(${SWIG_USE_FILE})

# When set (e.g. net10.0) the C# sources are built with the .NET SDK (dotnet),
# otherwise with csc/mcs targeting .NET Framework/Mono.
SET (CSHARP_TARGET_FRAMEWORK "" CACHE STRING "Target framework (e.g. net10.0) used to build the C# sources with the .NET SDK. Empty uses csc/mcs.")

IF (CSHARP_TARGET_FRAMEWORK)
	FIND_PROGRAM (DOTNET_EXECUTABLE NAMES dotnet)
	IF (DOTNET_EXECUTABLE)
		MESSAGE(STATUS "Found .NET SDK: ${DOTNET_EXECUTABLE}, building C# for ${CSHARP_TARGET_FRAMEWORK}")
	ELSE (DOTNET_EXECUTABLE)
		MESSAGE(FATAL_ERROR "Could not find dotnet, required by CSHARP_TARGET_FRAMEWORK=${CSHARP_TARGET_FRAMEWORK}")
	ENDIF (DOTNET_EXECUTABLE)
	MARK_AS_ADVANCED(DOTNET_EXECUTABLE)
ELSE (CSHARP_TARGET_FRAMEWORK)
	FIND_PROGRAM (CSHARP_COMPILER NAMES csc mcs gmcs gmcs2)

	IF (CSHARP_COMPILER)
		MESSAGE(STATUS "Found CSharp compiler: ${CSHARP_COMPILER}")
	ELSE (CSHARP_COMPILER)
		MESSAGE(FATAL_ERROR "Could not find csharp compiler")
	ENDIF (CSHARP_COMPILER)
ENDIF (CSHARP_TARGET_FRAMEWORK)

if (WIN32)
  if (NOT MSVC71)
    if (CMAKE_CL_64)
      set(PLATFORM_TARGET /platform:anycpu)
    else(CMAKE_CL_64)
      set(PLATFORM_TARGET /platform:x86)
    endif(CMAKE_CL_64)
	set (KEYFILE_SPEC /keyfile:${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\mapscript.snk)
  endif (NOT MSVC71)
  if (MSVC_VERSION GREATER 1500)
      set(PLATFORM_TARGET ${PLATFORM_TARGET} /define:CLR4)
  endif (MSVC_VERSION GREATER 1500)
  MESSAGE(STATUS "Cmake build type: ${CMAKE_BUILD_TYPE}")
  if (CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
      set(PLATFORM_TARGET ${PLATFORM_TARGET} /debug:full)
  endif (CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
else()
	set (KEYFILE_SPEC -keyfile:${PROJECT_SOURCE_DIR}/src/mapscript/csharp/mapscript.snk)
endif(WIN32)

MARK_AS_ADVANCED(CSHARP_COMPILER)

include_directories(${PROJECT_SOURCE_DIR}/src/mapscript/swiginc)
include_directories(${PROJECT_SOURCE_DIR}/src/mapscript/)
include_directories(${PROJECT_SOURCE_DIR}/src/mapscript/csharp)
include_directories(${PROJECT_BINARY_DIR}/src/mapscript/)
SET (CMAKE_SWIG_OUTDIR "${CMAKE_CURRENT_BINARY_DIR}")
if (WIN32)
	SET( CMAKE_SWIG_FLAGS -namespace OSGeo.MapServer -DWIN32 ${MAPSERVER_COMPILE_DEFINITIONS})
else()
	SET( CMAKE_SWIG_FLAGS -namespace OSGeo.MapServer ${MAPSERVER_COMPILE_DEFINITIONS})
endif()

SWIG_ADD_LIBRARY(mapscript LANGUAGE csharp SOURCES ../mapscript.i)

# CMake 3.29.0 corrects an issue with earlier CMake versions regarding the
# shared library name (cf https://gitlab.kitware.com/cmake/cmake/-/issues/25405
# / https://gitlab.kitware.com/cmake/cmake/-/merge_requests/8973)
if (WIN32 OR (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.29.0"))
	set_target_properties(mapscript PROPERTIES OUTPUT_NAME "mapscript")
else()
	set_target_properties(mapscript PROPERTIES OUTPUT_NAME "libmapscript")
endif()

target_link_libraries(mapscript PRIVATE ${MAPSERVER_LIBMAPSERVER})




if (CSHARP_TARGET_FRAMEWORK)

	# Every project gets its own folder under dotnet/ and the build results are
	# copied next to the SWIG generated sources, where the csc build puts them.
	set(DOTNET_PROJECTS_DIR ${CMAKE_CURRENT_BINARY_DIR}/dotnet)
	set(DOTNET_BUILD_OPTIONS -c $<IF:$<CONFIG:Debug>,Debug,Release> --nologo)

	configure_file(${CMAKE_CURRENT_SOURCE_DIR}/mapscript_csharp.csproj.in ${DOTNET_PROJECTS_DIR}/mapscript_csharp/mapscript_csharp.csproj @ONLY)
	set(DOTNET_COMMANDS
		COMMAND ${DOTNET_EXECUTABLE} build ${DOTNET_PROJECTS_DIR}/mapscript_csharp/mapscript_csharp.csproj ${DOTNET_BUILD_OPTIONS} -o ${DOTNET_PROJECTS_DIR}/mapscript_csharp/out
		COMMAND ${CMAKE_COMMAND} -E copy_if_different ${DOTNET_PROJECTS_DIR}/mapscript_csharp/out/mapscript_csharp.dll ${DOTNET_PROJECTS_DIR}/mapscript_csharp/out/mapscript_csharp.pdb ${CMAKE_CURRENT_BINARY_DIR}
	)

	set(EXAMPLES shpdump drawmap shapeinfo drawquery HTMLtemplate RFC24)
	# System.Drawing is only supported on Windows in .NET
	set(DRAWING_EXAMPLES getbytes drawmapDirect drawmapDirectPrint drawmapGDIPlus)
	if (WIN32)
		list(APPEND EXAMPLES ${DRAWING_EXAMPLES})
	endif (WIN32)
	foreach (EXAMPLE_NAME ${EXAMPLES})
		if (EXAMPLE_NAME IN_LIST DRAWING_EXAMPLES)
			set(EXAMPLE_TARGET_FRAMEWORK ${CSHARP_TARGET_FRAMEWORK}-windows)
			set(EXAMPLE_USE_WINDOWS_FORMS true)
		else ()
			set(EXAMPLE_TARGET_FRAMEWORK ${CSHARP_TARGET_FRAMEWORK})
			set(EXAMPLE_USE_WINDOWS_FORMS false)
		endif ()
		set(EXAMPLE_DIR ${DOTNET_PROJECTS_DIR}/${EXAMPLE_NAME})
		configure_file(${CMAKE_CURRENT_SOURCE_DIR}/example.csproj.in ${EXAMPLE_DIR}/${EXAMPLE_NAME}.csproj @ONLY)
		list(APPEND DOTNET_COMMANDS
			COMMAND ${DOTNET_EXECUTABLE} build ${EXAMPLE_DIR}/${EXAMPLE_NAME}.csproj ${DOTNET_BUILD_OPTIONS} -o ${EXAMPLE_DIR}/out
			COMMAND ${CMAKE_COMMAND} -E copy_if_different ${EXAMPLE_DIR}/out/${EXAMPLE_NAME}${CMAKE_EXECUTABLE_SUFFIX} ${EXAMPLE_DIR}/out/${EXAMPLE_NAME}.dll ${EXAMPLE_DIR}/out/${EXAMPLE_NAME}.runtimeconfig.json ${EXAMPLE_DIR}/out/${EXAMPLE_NAME}.deps.json ${CMAKE_CURRENT_BINARY_DIR}
		)
	endforeach ()

	ADD_CUSTOM_COMMAND(TARGET mapscript
						WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
						POST_BUILD
						${DOTNET_COMMANDS}
						COMMENT "Compiling c# source files for ${CSHARP_TARGET_FRAMEWORK}"
						)
elseif (WIN32)

	ADD_CUSTOM_COMMAND(TARGET mapscript
						  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
						  POST_BUILD
						  COMMAND set MAPSCRIPT_SNK=${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\mapscript.snk
						  COMMAND set MAPSCRIPT_SNK=%MAPSCRIPT_SNK:/=\\%
						  COMMAND copy /Y \"%MAPSCRIPT_SNK%\"
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /t:library /out:mapscript_csharp.dll ${KEYFILE_SPEC} *.cs ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\config\\AssemblyInfo.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:shpdump.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\shpdump.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:drawmap.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmap.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:shapeinfo.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\shapeinfo.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:drawquery.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawquery.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /r:System.Drawing.dll /out:getbytes.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\getbytes.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:HTMLtemplate.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\HTMLtemplate.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /out:RFC24.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\RFC24.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /r:System.Drawing.dll /out:drawmapDirect.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmapDirect.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /r:System.Drawing.dll /out:drawmapDirectPrint.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmapDirectPrint.cs
						  COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /r:mapscript_csharp.dll /r:System.Drawing.dll /out:drawmapGDIPlus.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmapGDIPlus.cs
						  #COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /lib:C:\Windows\Microsoft.NET\Framework\v4.0.30319\WPF /r:mapscript_csharp.dll /r:System.Xaml.dll /r:WindowsBase.dll /r:PresentationCore.dll /out:drawmapWPF.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmapWPF.cs
						  COMMENT "Compiling c# source files"
						  )
else()
	ADD_CUSTOM_COMMAND(TARGET mapscript
						WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
						POST_BUILD
						COMMAND cp -f ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/mapscript.snk ./
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -target:library -out:mapscript_csharp.dll ${KEYFILE_SPEC} *.cs ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/config/AssemblyInfo.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:shpdump.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/shpdump.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:drawmap.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/drawmap.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:shapeinfo.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/shapeinfo.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:drawquery.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/drawquery.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -reference:System.Drawing.dll -out:getbytes.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/getbytes.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:HTMLtemplate.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/HTMLtemplate.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -out:RFC24.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/RFC24.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -reference:System.Drawing.dll -out:drawmapDirect.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/drawmapDirect.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -reference:System.Drawing.dll -out:drawmapDirectPrint.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/drawmapDirectPrint.cs
						COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} -reference:mapscript_csharp.dll -reference:System.Drawing.dll -out:drawmapGDIPlus.exe ${PROJECT_SOURCE_DIR}/src/mapscript/csharp/examples/drawmapGDIPlus.cs
						#COMMAND ${CSHARP_COMPILER} ${PLATFORM_TARGET} /lib:C:\Windows\Microsoft.NET\Framework\v4.0.30319\WPF /r:mapscript_csharp.dll /r:System.Xaml.dll /r:WindowsBase.dll /r:PresentationCore.dll /out:drawmapWPF.exe ${PROJECT_SOURCE_DIR}\\src\\mapscript\\csharp\\examples\\drawmapWPF.cs
						COMMENT "Compiling c# source files"
						)
endif()


