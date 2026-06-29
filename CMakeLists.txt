cmake_minimum_required(VERSION 3.14)
project(string_metrics LANGUAGES CXX)

include(FetchContent)

# --- Dependency: strings ---
# basic_string_metric.h includes "strings.hh" for strings::upper (its
# case-folding step). It lives in the sibling strings library, pulled in here so
# the include resolves. We track its tip with GIT_TAG main, like the rest of the
# family. We declare it before phonetic so this declaration wins: phonetic also
# FetchContent_Declare's strings, and FetchContent dedups by the declared name,
# keeping the first declaration.
FetchContent_Declare(
  strings
  GIT_REPOSITORY https://github.com/Kronuz/strings.git
  GIT_TAG        main
)

# --- Dependency: phonetic ---
# soundex_metric.h is a Soundex-backed metric: it encodes both inputs with a
# phonetic encoder, then defers to an inner string metric. The concrete encoders
# (SoundexEnglish in "english_soundex.h", etc.) live in the sibling phonetic
# library, pulled in here so those includes resolve for consumers (and our test /
# demo). phonetic itself depends on strings; declaring strings above makes that
# transitive dependency resolve to one copy.
FetchContent_Declare(
  phonetic
  GIT_REPOSITORY https://github.com/Kronuz/phonetic.git
  GIT_TAG        main
)

FetchContent_MakeAvailable(strings phonetic)

# --- Library target ---
# string-metrics is header-only: the metrics are a CRTP base (StringMetric<Impl>)
# plus per-algorithm subclasses, all templates / inline in headers. So the target
# is a pure INTERFACE library that just puts the source dir on the consumer's
# include path.
add_library(string_metrics INTERFACE)
target_include_directories(string_metrics INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}")
target_compile_features(string_metrics INTERFACE cxx_std_20)

# Link strings and phonetic INTERFACE so "strings.hh" and the soundex headers
# ride onto the consumer's include path: both are part of the public surface
# (basic_string_metric.h includes strings.hh; a consumer of soundex_metric.h
# instantiates it with a phonetic encoder), so a consumer needs them resolvable.
target_link_libraries(string_metrics INTERFACE strings::strings phonetic::phonetic)

# Alias so consumers can link `string_metrics::string_metrics` whether they add
# this via add_subdirectory or FetchContent.
add_library(string_metrics::string_metrics ALIAS string_metrics)

# Only build the test and demo when this repo is the top-level project, so
# consumers (e.g. via FetchContent) don't build them.
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  enable_testing()
  add_executable(string_metrics_test test/test.cc)
  target_link_libraries(string_metrics_test PRIVATE string_metrics)
  add_test(NAME string_metrics COMMAND string_metrics_test)

  # A runnable tour of the library (not a test): build & run ./build/string_metrics_demo
  add_executable(string_metrics_demo examples/demo.cc)
  target_link_libraries(string_metrics_demo PRIVATE string_metrics)
endif()
