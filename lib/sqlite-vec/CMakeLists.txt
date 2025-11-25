# sqlite-vec - Vector Search Extension for SQLite
# Repository: https://github.com/asg017/sqlite-vec
# License: Apache 2.0 / MIT

set(SQLITE_VEC_AVAILABLE OFF)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/sqlite-vec.c" AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/sqlite-vec.h")
  set(SQLITE_VEC_AVAILABLE ON)
endif()

if(SQLITE_VEC_AVAILABLE)
  set(SOURCES sqlite-vec.c)
  set(HEADERS sqlite-vec.h)

  add_library(sqlite-vec STATIC ${SOURCES})

  target_include_directories(sqlite-vec SYSTEM INTERFACE ${CMAKE_CURRENT_SOURCE_DIR})
  target_include_directories(sqlite-vec PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

  find_package(SQLite3 REQUIRED)
  target_link_libraries(sqlite-vec PRIVATE SQLite3::SQLite3)

  target_compile_definitions(sqlite-vec PRIVATE
    SQLITE_CORE=1
    SQLITE_VEC_ENABLE_AVX2=1
    SQLITE_VEC_ENABLE_NEON=1
  )

  set_target_properties(sqlite-vec PROPERTIES
    CXX_CLANG_TIDY ""
    CXX_CPPCHECK ""
    CXX_INCLUDE_WHAT_YOU_USE ""
    FOLDER lib
  )

  source_group_by_folder(sqlite-vec)

  target_compile_definitions(sqlite-vec INTERFACE HAVE_SQLITE_VEC=1)
else()
  message(WARNING "sqlite-vec sources not found. Vector similarity search will be disabled.")
  add_library(sqlite-vec INTERFACE)
  target_compile_definitions(sqlite-vec INTERFACE HAVE_SQLITE_VEC=0)
endif()

target_link_libraries(lib${APP_NAME_LC} PUBLIC sqlite-vec)
