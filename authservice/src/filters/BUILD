load("//bazel:bazel.bzl", "authsvc_cc_library")

package(default_visibility = ["//visibility:public"])

authsvc_cc_library(
    name = "filter",
    srcs = [],
    hdrs = ["filter.h"],
    deps = [
        "@boost//:asio",
        "@boost//:coroutine",
        "@com_github_abseil-cpp//absl/strings:strings",
        "@com_google_googleapis//google/rpc:code_cc_proto",
        "@envoy_api//envoy/service/auth/v3:pkg_cc_grpc",
    ],
)

authsvc_cc_library(
    name = "pipe",
    srcs = ["pipe.cc"],
    hdrs = ["pipe.h"],
    deps = [
        ":filter",
        "@com_github_grpc_grpc//:grpc++",
    ],
)

authsvc_cc_library(
    name = "filter_factory",
    hdrs = ["filter_factory.h"],
    deps = [
        ":pipe",
    ],
)

authsvc_cc_library(
    name = "filter_chain",
    srcs = [
        "filter_chain.cc",
    ],
    hdrs = [
        "filter_chain.h",
    ],
    deps = [
        ":filter_factory",
        "//config:config_cc",
        "//src/config",
        "//src/filters:filter",
        "//src/filters:pipe",
        "//src/filters/mock:mock_filter",
        "//src/filters/oidc:oidc_filter",
        "//src/filters/oidc:redis_session_store",
        "//src/filters/oidc:redis_wrapper",
        "@com_github_abseil-cpp//absl/strings:strings",
        "@com_github_gabime_spdlog//:spdlog",
        "@com_github_sewenew_redis_plus_plus//:redis_plus_plus",
        "@envoy_api//envoy/service/auth/v3:pkg_cc_grpc",
    ],
)
