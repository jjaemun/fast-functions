# --testing.

include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG main
)

set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_executable(
    memtests
    memtests.cpp
)

target_link_libraries(
    memtests PRIVATE
    GTest::gtest_main
    ffn
)

include(GoogleTest)
gtest_discover_tests(memtests)
