# zephyr_library()


#---
# Patch 1: Skip CONFIGURATION_RESET on startup (causes I2C issues on nRF52840)
# Patch 2: Accumulative movement filter to prevent auto-mouse false triggers
#          from typing vibration while preserving smooth trackball movement.
#          - Accumulates small movements, reports when threshold exceeded
#          - Resets accumulator if no movement for 50ms (typing gap)
#          - Trackball data arrives every ~8ms, so accumulation is fast
#          - Threshold 7 is scaled for CPI=1200 (default)
#---
set(FILE_PATH "${ZEPHYR_BASE}/../zmk-driver-pat9125/src/pat9125.c")
file(READ "${FILE_PATH}" FILE_CONTENT)

# Patch 1: Skip CONFIGURATION_RESET write
string(
  REPLACE "\ti2c_reg_write_byte_dt(&cfg->i2c, PAT9125_CONFIGURATION, CONFIGURATION_RESET);\n"
          "\t\/\/ i2c_reg_write_byte_dt(&cfg->i2c, PAT9125_CONFIGURATION, CONFIGURATION_RESET);\n"
  FILE_CONTENT "${FILE_CONTENT}"
)

# Patch 2a: Add accumulator variables
string(
  REPLACE "static void pat9125_motion_work_handler(struct k_work *work)\n{"
          "static int pat9125_acc_x = 0;\nstatic int pat9125_acc_y = 0;\nstatic int64_t pat9125_acc_last_time = 0;\n\nstatic void pat9125_motion_work_handler(struct k_work *work)\n{"
  FILE_CONTENT "${FILE_CONTENT}"
)

# Patch 2b: Add accumulator logic before input_report_rel
string(
  REPLACE "\tLOG_DBG(\"x=%4d y=%4d\", x, y);\n\n\tif (cfg->axis_x >= 0) {"
          "\tLOG_DBG(\"x=%4d y=%4d\", x, y);\n\n\t\/\/ Accumulative movement filter (threshold=7, timeout=50ms)\n\t{\n\t\tint64_t now = k_uptime_get();\n\t\tif (now - pat9125_acc_last_time > 50) {\n\t\t\tpat9125_acc_x = 0;\n\t\t\tpat9125_acc_y = 0;\n\t\t}\n\t\tpat9125_acc_last_time = now;\n\t\tpat9125_acc_x += x;\n\t\tpat9125_acc_y += y;\n\t\tif (abs(pat9125_acc_x) + abs(pat9125_acc_y) <= 7) {\n\t\t\tk_work_submit(&data->motion_work);\n\t\t\treturn;\n\t\t}\n\t\tx = pat9125_acc_x;\n\t\ty = pat9125_acc_y;\n\t\tpat9125_acc_x = 0;\n\t\tpat9125_acc_y = 0;\n\t}\n\n\tif (cfg->axis_x >= 0) {"
  FILE_CONTENT "${FILE_CONTENT}"
)

file(WRITE "${FILE_PATH}" "${FILE_CONTENT}")
message(STATUS "Patched PAT9125 driver (reset skip + movement accumulator): ${FILE_PATH}")


# zephyr_include_directories(include)
