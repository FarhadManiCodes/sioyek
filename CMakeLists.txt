cmake_minimum_required(VERSION 3.16)
project(sioyek VERSION 2.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(SIOYEK_NO_TTS "Build without Qt TextToSpeech (no read-aloud)" OFF)

# Qt Stuff
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

set(_sioyek_qt_components Core Gui Widgets Network OpenGL QuickWidgets Svg)
if(NOT SIOYEK_NO_TTS)
  list(APPEND _sioyek_qt_components TextToSpeech)
endif()

find_package(QT NAMES Qt6 Qt5 REQUIRED COMPONENTS Core)
if(QT_VERSION_MAJOR GREATER_EQUAL 6)
  find_package(Qt6 REQUIRED COMPONENTS
        ${_sioyek_qt_components} OpenGLWidgets
  )
else()
  find_package(Qt5 REQUIRED COMPONENTS
        ${_sioyek_qt_components} OpenGLExtensions
  )
endif()

qt_standard_project_setup()

qt_add_executable(sioyek WIN32 MACOSX_BUNDLE
    resources.qrc
    pdf_viewer/OpenWithApplication.cpp pdf_viewer/OpenWithApplication.h
    pdf_viewer/RunGuard.cpp pdf_viewer/RunGuard.h
    pdf_viewer/book.cpp pdf_viewer/book.h
    pdf_viewer/checksum.cpp pdf_viewer/checksum.h
    pdf_viewer/config.cpp pdf_viewer/config.h
    pdf_viewer/coordinates.cpp pdf_viewer/coordinates.h
    pdf_viewer/database.cpp pdf_viewer/database.h
    pdf_viewer/document.cpp pdf_viewer/document.h
    pdf_viewer/document_view.cpp pdf_viewer/document_view.h
    pdf_viewer/fts_fuzzy_match.h
    pdf_viewer/input.cpp pdf_viewer/input.h
    pdf_viewer/main.cpp
    pdf_viewer/main_widget.cpp pdf_viewer/main_widget.h
    pdf_viewer/mysortfilterproxymodel.cpp pdf_viewer/mysortfilterproxymodel.h
    pdf_viewer/new_file_checker.cpp pdf_viewer/new_file_checker.h
    pdf_viewer/path.cpp pdf_viewer/path.h
    pdf_viewer/pdf_renderer.cpp pdf_viewer/pdf_renderer.h
    pdf_viewer/pdf_view_opengl_widget.cpp pdf_viewer/pdf_view_opengl_widget.h
    pdf_viewer/rapidfuzz_amalgamated.hpp
    pdf_viewer/touchui/TouchAudioButtons.cpp pdf_viewer/touchui/TouchAudioButtons.h
    pdf_viewer/touchui/TouchCheckbox.cpp pdf_viewer/touchui/TouchCheckbox.h
    pdf_viewer/touchui/TouchConfigMenu.cpp pdf_viewer/touchui/TouchConfigMenu.h
    pdf_viewer/touchui/TouchCopyOptions.cpp pdf_viewer/touchui/TouchCopyOptions.h
    pdf_viewer/touchui/TouchDeleteButton.cpp pdf_viewer/touchui/TouchDeleteButton.h
    pdf_viewer/touchui/TouchDrawControls.cpp pdf_viewer/touchui/TouchDrawControls.h
    pdf_viewer/touchui/TouchGenericButtons.cpp pdf_viewer/touchui/TouchGenericButtons.h
    pdf_viewer/touchui/TouchHighlightButtons.cpp pdf_viewer/touchui/TouchHighlightButtons.h
    pdf_viewer/touchui/TouchListView.cpp pdf_viewer/touchui/TouchListView.h
    pdf_viewer/touchui/TouchMacroEditor.cpp pdf_viewer/touchui/TouchMacroEditor.h
    pdf_viewer/touchui/TouchMainMenu.cpp pdf_viewer/touchui/TouchMainMenu.h
    pdf_viewer/touchui/TouchMarkSelector.cpp pdf_viewer/touchui/TouchMarkSelector.h
    pdf_viewer/touchui/TouchPageSelector.cpp pdf_viewer/touchui/TouchPageSelector.h
    pdf_viewer/touchui/TouchRangeSelectUI.cpp pdf_viewer/touchui/TouchRangeSelectUI.h
    pdf_viewer/touchui/TouchRectangleSelectUI.cpp pdf_viewer/touchui/TouchRectangleSelectUI.h
    pdf_viewer/touchui/TouchSearchButtons.cpp pdf_viewer/touchui/TouchSearchButtons.h
    pdf_viewer/touchui/TouchSettings.cpp pdf_viewer/touchui/TouchSettings.h
    pdf_viewer/touchui/TouchSlider.cpp pdf_viewer/touchui/TouchSlider.h
    pdf_viewer/touchui/TouchTextEdit.cpp pdf_viewer/touchui/TouchTextEdit.h
    pdf_viewer/ui.cpp pdf_viewer/ui.h
    pdf_viewer/utf8.h
    pdf_viewer/utf8/checked.h
    pdf_viewer/utf8/core.h
    pdf_viewer/utf8/unchecked.h
    pdf_viewer/utils.cpp pdf_viewer/utils.h
)

if(APPLE)
  target_sources(sioyek PRIVATE
        pdf_viewer/macos_specific.mm
  )
endif()

target_compile_options(sioyek PRIVATE -Wno-inconsistent-missing-override)

target_include_directories(sioyek PRIVATE 
    pdf_viewer 
    mupdf/include
    ${SQLite3_INCLUDE_DIRS}
)

target_compile_definitions(sioyek PRIVATE
    QT_OPENGLEXTENSIONS_LIB
    QT_OPENGL_LIB
    QT_WIDGETS_LIB
)

target_link_libraries(sioyek PRIVATE
    Qt::Core
    Qt::Gui
    Qt::Network
    Qt::OpenGL
    Qt::QuickWidgets
    Qt::Svg
    Qt::Widgets
)

if(NOT SIOYEK_NO_TTS)
  target_link_libraries(sioyek PRIVATE Qt::TextToSpeech)
else()
  target_compile_definitions(sioyek PRIVATE SIOYEK_NO_TTS)
endif()

if(NOT ANDROID)
  target_sources(sioyek PUBLIC
        pdf_viewer/synctex/synctex_parser.c pdf_viewer/synctex/synctex_parser.h
        pdf_viewer/synctex/synctex_parser_utils.c pdf_viewer/synctex/synctex_parser_utils.h
  )
else()
  target_compile_definitions(sioyek PRIVATE
        SIOYEK_ANDROID
  )
endif()

if(WIN32)
  target_compile_definitions(sioyek PRIVATE
        _CRT_SECURE_NO_WARNINGS
        _CRT_NONSTDC_NO_DEPRECATE
        NOMINMAX
  )

  if(MSVC)
    target_compile_options(sioyek PRIVATE /bigobj /MP)
  endif()
endif()

# Note: upstream CMakeLists.txt defines LINUX_STANDARD_PATHS unconditionally on
# Linux, which forces sioyek to look at /etc/sioyek and /usr/share/sioyek for
# defaults instead of files next to the binary. That makes the CMake build
# incompatible with portable layouts. The qmake build's "linux_app_image"
# config omits this define for the same reason. We omit it here too so this
# build can run from ~/.local/share/sioyek/ without a system install.

find_package(ZLIB REQUIRED)
find_package(HarfBuzz REQUIRED)
find_package(SQLite3 REQUIRED)

if((QT_VERSION_MAJOR GREATER 5))
  target_compile_definitions(sioyek PRIVATE
        SIOYEK_QT6
  )

  target_link_libraries(sioyek PRIVATE
        Qt::OpenGLWidgets
  )
endif()

add_library(fzf STATIC
    fzf/fzf.c
    fzf/fzf.h
)

target_link_libraries(sioyek PRIVATE
    ${CMAKE_DL_LIBS}
    ${HARFBUZZ_LIBRARIES}
    ZLIB::ZLIB
    fzf
    SQLite3::SQLite3
)

if(APPLE)
  target_link_libraries(sioyek PRIVATE
      "-framework AppKit"
      -ldl
      -L${CMAKE_CURRENT_SOURCE_DIR}/mupdf/build/release
      -lmupdf
      -lmupdf-third
      -lmupdf-threads
      -lz
  )
else()
  string(REPLACE "-mno-direct-extern-access" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  add_compile_options(-fvisibility=hidden -fvisibility-inlines-hidden)
  add_link_options(-Wl,-O2)
  target_link_libraries(sioyek PRIVATE
      -L${CMAKE_CURRENT_SOURCE_DIR}/mupdf/build/release
      -lmupdf
      -lmupdf-third
      -lmupdf-threads
      -lharfbuzz
      -lfreetype
  )
endif()

install(TARGETS sioyek
    BUNDLE DESTINATION .
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

qt_generate_deploy_app_script(
    TARGET sioyek
    OUTPUT_SCRIPT deploy_script
    NO_UNSUPPORTED_PLATFORM_ERROR
)
install(SCRIPT ${deploy_script})
