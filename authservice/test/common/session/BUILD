load("//bazel:bazel.bzl", "authsvc_cc_library", "authsvc_cc_test")

authsvc_cc_library(
    name = "mocks",
    hdrs = ["mocks.h"],
    visibility = ["//test:__subpackages__"],
    deps = [
        "//src/common/session:session_string_generator",
    ],
)

authsvc_cc_test(
    name = "session_string_generator_test",
    srcs = ["session_string_generator_test.cc"],
    deps = [
        "//src/common/session:session_string_generator",
        "@com_google_googletest//:gtest_main",
    ],
)
