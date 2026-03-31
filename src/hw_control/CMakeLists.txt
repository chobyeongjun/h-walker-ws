add_library(hw_control INTERFACE)
target_include_directories(hw_control INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(hw_control INTERFACE hw_common)
