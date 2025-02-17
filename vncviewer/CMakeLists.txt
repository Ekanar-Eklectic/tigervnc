include_directories(${FLTK_INCLUDE_DIR})
include_directories(${GETTEXT_INCLUDE_DIR})

include_directories(${CMAKE_SOURCE_DIR}/common)
set(VNCVIEWER_SOURCES
  menukey.cxx
  BaseTouchHandler.cxx
  CConn.cxx
  DesktopWindow.cxx
  EmulateMB.cxx
  UserDialog.cxx
  ServerDialog.cxx
  Surface.cxx
  OptionsDialog.cxx
  PlatformPixelBuffer.cxx
  Viewport.cxx
  parameters.cxx
  keysym2ucs.c
  touch.cxx
  MonitorArrangement.cxx
  MonitorIndicesParameter.cxx
  vncviewer.cxx)

if(WIN32)
  # Since vncviewer.rc is generated, local includes will be looking
  # in the wrong directory. We need to help it out.
  include_directories(${CMAKE_CURRENT_SOURCE_DIR})
  configure_file(vncviewer.rc.in vncviewer.rc)
  set(VNCVIEWER_SOURCES
    ${VNCVIEWER_SOURCES}
    ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.rc)
endif()

if(WIN32)
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} Win32TouchHandler.cxx win32.c)
elseif(APPLE)
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} cocoa.mm osx_to_qnum.c)
else()
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} GestureHandler.cxx XInputTouchHandler.cxx xkb_to_qnum.c)
endif()

if(WIN32)
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} Surface_Win32.cxx)
elseif(APPLE)
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} Surface_OSX.cxx)
else()
  set(VNCVIEWER_SOURCES ${VNCVIEWER_SOURCES} Surface_X11.cxx)
endif()

if(WIN32 AND NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
  add_executable(vncviewer WIN32 ${VNCVIEWER_SOURCES})
else()
  add_executable(vncviewer ${VNCVIEWER_SOURCES})
endif()

target_link_libraries(vncviewer rfb network rdr os ${FLTK_LIBRARIES} ${GETTEXT_LIBRARIES} ${H264_LIBRARIES})

if(WIN32)
  target_link_libraries(vncviewer msimg32)
elseif(APPLE)
  target_link_libraries(vncviewer "-framework Cocoa")
  target_link_libraries(vncviewer "-framework Carbon")
  target_link_libraries(vncviewer "-framework IOKit")
else()
  target_link_libraries(vncviewer ${X11_Xi_LIB})

  if(X11_Xrandr_LIB)
    add_definitions(-DHAVE_XRANDR)
    target_link_libraries(vncviewer ${X11_Xrandr_LIB})
  endif()
endif()

install(TARGETS vncviewer DESTINATION ${CMAKE_INSTALL_FULL_BINDIR})
if(UNIX)
  install(FILES vncviewer.man DESTINATION ${CMAKE_INSTALL_FULL_MANDIR}/man1 RENAME vncviewer.1)

  configure_file(vncviewer.desktop.in.in vncviewer.desktop.in)
  find_program(INTLTOOL_MERGE_EXECUTABLE intltool-merge)
  if("${GETTEXT_VERSION_STRING}" VERSION_GREATER 0.18.99)
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND ${GETTEXT_MSGFMT_EXECUTABLE}
                --desktop --template vncviewer.desktop.in
                -d ${CMAKE_SOURCE_DIR}/po -o vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
    )
  elseif(INTLTOOL_MERGE_EXECUTABLE)
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND sed -e 's/^Name/_Name/'
                -e 's/^GenericName/_GenericName/'
                -e 's/^Comment/_Comment/'
                vncviewer.desktop.in > vncviewer.desktop.intl
      COMMAND ${INTLTOOL_MERGE_EXECUTABLE}
                -d ${CMAKE_SOURCE_DIR}/po
                vncviewer.desktop.intl vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
    )
  else()
    add_custom_command(OUTPUT vncviewer.desktop
      COMMAND cp vncviewer.desktop.in vncviewer.desktop
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop.in
    )
  endif()
  add_custom_target(desktop ALL DEPENDS vncviewer.desktop)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/vncviewer.desktop DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/applications)

  foreach(res 16 22 24 32 48)
    install(FILES ../media/icons/tigervnc_${res}.png DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/icons/hicolor/${res}x${res}/apps RENAME tigervnc.png)
  endforeach()
  install(FILES ../media/icons/tigervnc.svg DESTINATION ${CMAKE_INSTALL_FULL_DATADIR}/icons/hicolor/scalable/apps)
endif()
