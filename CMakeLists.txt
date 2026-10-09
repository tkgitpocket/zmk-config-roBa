# DYA Studio対応: zmk-feature-device-info が使う <zephyr/version.h> の互換シム
# （include/zephyr/version.h）をグローバルなインクルードパスに追加する。
zephyr_include_directories(${CMAKE_CURRENT_LIST_DIR}/include)
