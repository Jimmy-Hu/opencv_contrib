set(test_deps opencv_cudev opencv_core opencv_imgproc opencv_imgcodecs opencv_videoio opencv_highgui opencv_ts ${OPENCV_MODULE_opencv_ts_DEPS})

ocv_check_dependencies(${test_deps})

if(OCV_DEPENDENCIES_FOUND)
  set(the_target "opencv_test_${name}")

  ocv_module_include_directories("${test_deps}" "${the_module}")

  file(GLOB test_srcs "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/*.cu")
  file(GLOB test_hdrs "${CMAKE_CURRENT_SOURCE_DIR}/*.hpp")
  source_group("Src" FILES ${test_srcs})
  source_group("Include" FILES ${test_hdrs})
  set(OPENCV_TEST_${the_module}_SOURCES ${test_srcs} ${test_hdrs})

  ocv_cuda_filter_options()

  set(target_libs ${test_deps} ${OPENCV_LINKER_LIBS})
  if(NOT ENABLE_CUDA_FIRST_CLASS_LANGUAGE)
    ocv_check_windows_crt_linkage()
    set(target_libs ${target_libs} ${CUDA_LIBRARIES})
    set(test_cudev_cuda_options "")
    if(CUDA_VERSION VERSION_LESS "11.0")
      # Windows version does not support --std option
      if(UNIX OR APPLE)
        list(APPEND test_cudev_cuda_options "-std=c++11")
      endif()
    else()
      if(CUDA_VERSION VERSION_LESS "12.8")
        list(APPEND test_cudev_cuda_options "-std=c++14")
      else()
        list(APPEND test_cudev_cuda_options "-std=c++17")
        if(WIN32)
          list(APPEND test_cudev_cuda_options "-Xcompiler=/Zc:preprocessor")
        endif()
      endif()
      ocv_warnings_disable(CMAKE_CXX_FLAGS -Wdeprecated-declarations)
    endif()
    CUDA_ADD_EXECUTABLE(${the_target} ${OPENCV_TEST_${the_module}_SOURCES} OPTIONS ${test_cudev_cuda_options})
  else()
    ocv_add_executable(${the_target} ${OPENCV_TEST_${the_module}_SOURCES})
  endif()

  ocv_target_link_libraries(${the_target} PRIVATE ${target_libs})

  add_dependencies(opencv_tests ${the_target})

  set_target_properties(${the_target} PROPERTIES LABELS "${OPENCV_MODULE_${the_module}_LABEL}")
  set_source_files_properties(${OPENCV_TEST_${the_module}_SOURCES} ${${the_target}_pch}
    PROPERTIES LABELS "${OPENCV_MODULE_${the_module}_LABEL};AccuracyTest")

  # Additional target properties
  set_target_properties(${the_target} PROPERTIES
    DEBUG_POSTFIX "${OPENCV_DEBUG_POSTFIX}"
    RUNTIME_OUTPUT_DIRECTORY "${EXECUTABLE_OUTPUT_PATH}"
  )

  if(ENABLE_SOLUTION_FOLDERS)
    set_target_properties(${the_target} PROPERTIES FOLDER "tests accuracy")
  endif()

  ocv_add_test_from_target("${the_target}" "Accuracy" "${the_target}")

  if(INSTALL_TESTS)
    install(TARGETS ${the_target} RUNTIME DESTINATION ${OPENCV_TEST_INSTALL_PATH} COMPONENT tests)
  endif()
endif()
