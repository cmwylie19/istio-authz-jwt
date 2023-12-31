load("//bazel:bazel.bzl", "authsvc_cc_library", "authsvc_cc_test")

authsvc_cc_library(
    name = "mocks",
    hdrs = ["mocks.h"],
    visibility = ["//visibility:public"],
    deps = [
        "//src/filters/oidc:authorization_state",
        "//src/filters/oidc:jwks_resolver",
        "//src/filters/oidc:redis_wrapper",
        "//src/filters/oidc:session_store",
        "//src/filters/oidc:token_response",
        "@com_github_abseil-cpp//absl/strings:strings",
        "@com_github_abseil-cpp//absl/types:optional",
        "@com_github_sewenew_redis_plus_plus//:redis_plus_plus",
    ],
)

authsvc_cc_test(
    name = "token_response_test",
    srcs = ["token_response_test.cc"],
    deps = [
        ":mocks",
        "//src/filters/oidc:token_response",
        "@com_github_grpc_grpc//:grpc++",
        "@com_google_googletest//:gtest_main",
    ],
)

authsvc_cc_test(
    name = "in_memory_session_store_test",
    srcs = ["in_memory_session_store_test.cc"],
    deps = [
        "//src/common/utilities:time_service",
        "//src/filters/oidc:in_memory_session_store",
        "//test/common/utilities:mocks",
        "@com_google_googletest//:gtest_main",
    ],
)

authsvc_cc_test(
    name = "redis_session_store_test",
    srcs = ["redis_session_store_test.cc"],
    deps = [
        "//src/common/utilities:time_service",
        "//src/filters/oidc:redis_session_store",
        "//test/common/utilities:mocks",
        "//test/filters/oidc:mocks",
        "//test/shared:shared_test_helpers",
        "@com_google_googletest//:gtest_main",
    ],
)

authsvc_cc_test(
    name = "redis_retry_wrapper_test",
    srcs = ["redis_retry_wrapper_test.cc"],
    deps = [
        "//src/filters/oidc:redis_retry_wrapper",
        "//test/filters/oidc:mocks",
        "//test/shared:shared_test_helpers",
        "@com_google_googletest//:gtest_main",
    ],
)

authsvc_cc_test(
    name = "oidc_filter_test",
    srcs = ["oidc_filter_test.cc"],
    deps = [
        "//src/filters/oidc:oidc_filter",
        "//test/common/http:mocks",
        "//test/common/session:mocks",
        "//test/filters/oidc:mocks",
        "@com_github_grpc_grpc//:grpc++",
        "@com_google_googletest//:gtest_main",
    ],
)

authsvc_cc_test(
    name = "jwks_resolver_test",
    srcs = ["jwks_resolver_test.cc"],
    deps = [
        "//src/filters/oidc:jwks_resolver",
        "//test/common/http:mocks",
        "@com_google_googletest//:gtest_main",
    ],
)
