## First start with some standard catkin stuff.
cmake_minimum_required(VERSION 2.8.7)
project(integrated_viewer)

execute_process(
  COMMAND rosversion -d
  OUTPUT_VARIABLE ROS_VERSION
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

find_package(catkin REQUIRED COMPONENTS
  cv_bridge
  image_transport
  roscpp
  sensor_msgs
  std_msgs
  runtime_manager
  points2image
  rviz
  )

## This plugin includes Qt widgets, so we must include Qt.
## We'll use the version that rviz used so they are compatible.
if(rviz_QT_VERSION VERSION_LESS "5")
  message(STATUS "Using Qt4 based on the rviz_QT_VERSION: ${rviz_QT_VERSION}")
  find_package(Qt4 ${rviz_QT_VERSION} EXACT REQUIRED QtCore QtGui)
  ## pull in all required include dirs, define QT_LIBRARIES, etc.
  include(${QT_USE_FILE})
else()
  message(STATUS "Using Qt5 based on the rviz_QT_VERSION: ${rviz_QT_VERSION}")
  find_package(Qt5 ${rviz_QT_VERSION} EXACT REQUIRED Core Widgets Quick)
  ## make target_link_libraries(${QT_LIBRARIES}) pull in all required dependencies
  set(QT_LIBRARIES Qt5::Widgets)
endif()

catkin_package(DEPENDS rviz)
include(FindPkgConfig)
find_package(Qt5Widgets)
find_package(Qt5Quick)
find_package(Qt5Core)
pkg_check_modules(OpenCV REQUIRED opencv)

set(CMAKE_CXX_FLAGS
  "-std=c++11 -Wall ${CMAKE_CXX_FLAGS} ${Qt5Widgets_EXECUTABLE_COMPILE_FLAGS} ${Qt5Quick_EXECUTABLE_COMPILE_FLAGS}"
  )

include_directories(${catkin_INCLUDE_DIRS}
	${OpenCV_INCLUDE_DIRS}
	${CMAKE_CURRENT_SOURCE_DIR}
	${CMAKE_CURRENT_BINARY_DIR}
	lib/
        )
link_directories(${catkin_LIBRARY_DIRS}
	${OpenCV_LIBRARY_DIRS})

## This setting causes Qt's "MOC" generation to happen automatically.
set(CMAKE_INCLUDE_CURRENT_DIR ON)
#set(CMAKE_AUTOMOC ON)

qt5_wrap_ui(UI_HEADERS
	node/traffic_light_plugin/form.ui
	node/image_viewer_plugin/image_viewer_form.ui
	)
qt5_wrap_cpp(MOC_FILES
  	node/traffic_light_plugin/traffic_light_plugin.h
  	node/image_viewer_plugin/image_viewer_plugin.h
	)

## I prefer the Qt signals and slots to avoid defining "emit", "slots",
## etc because they can conflict with boost signals, so define QT_NO_KEYWORDS here.
add_definitions(-DQT_NO_KEYWORDS)

## Here we specify the list of source files.
## The generated MOC files are included automatically as headers.
set(SRC_FILES
  lib/convert_image.h
  node/traffic_light_plugin/traffic_light_plugin.cpp
  node/image_viewer_plugin/image_viewer_plugin.cpp
  node/image_viewer_plugin/draw_rects.cpp
  node/image_viewer_plugin/draw_points.cpp
  ${MOC_FILES}
  ${UI_HEADERS})

add_library(${PROJECT_NAME} ${SRC_FILES})

set_target_properties(${PROJECT_NAME}
  PROPERTIES
  COMPILE_FLAGS
  "-DIMAGE_VIEWER_DEFAULT_IMAGE=${CMAKE_CURRENT_SOURCE_DIR}/node/image_viewer_plugin/autoware_logo.png"
)

target_link_libraries(${PROJECT_NAME} ${QT_LIBRARIES} ${catkin_LIBRARIES})

## Install rules

install(TARGETS
  ${PROJECT_NAME}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
add_dependencies(${PROJECT_NAME}
  runtime_manager_generate_messages_cpp
  cv_tracker_generate_messages_cpp
  points2image_generate_messages_cpp)

install(FILES
  plugin_description.xml
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION})

