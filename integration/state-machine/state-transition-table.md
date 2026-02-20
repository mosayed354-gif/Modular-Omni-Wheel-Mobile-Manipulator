# State Transition Table (Advanced)

This document defines the **event-driven state machine** for the autonomous mobile manipulator.
It coordinates **Camera/QR → Base Navigation → Arm Pick/Place** with safety, retries, and timeouts.

---

## 1) Definitions

### States
- **IDLE**: Waiting for start.
- **SCAN_QR**: Search for QR in the scanning zone.
- **DECODE_QR**: Decode/validate QR payload.
- **PLAN_MISSION**: Map QR → target zone + pick strategy.
- **NAVIGATE_TO_PICK**: Move base to pick area / object vicinity.
- **ALIGN_FOR_PICK**: Fine alignment (slow control).
- **ARM_PREPARE**: Move arm to pre-grasp pose.
- **GRASP**: Close gripper and confirm grip.
- **LIFT**: Lift object to safe transport height.
- **NAVIGATE_TO_PLACE**: Move base to destination zone.
- **ALIGN_FOR_PLACE**: Fine alignment at destination.
- **PLACE**: Release object.
- **RESET_ARM**: Return arm to safe/home pose.
- **DONE**: Task completed / wait for next.
- **ERROR**: Fault handling and safe stop.

### Events & Flags (Examples)
- `EV_START`
- `EV_QR_DETECTED`
- `EV_QR_DECODED_OK`
- `EV_QR_DECODED_FAIL`
- `EV_TARGET_REACHED`
- `EV_ALIGNED_OK`
- `EV_GRIP_OK`
- `EV_GRIP_FAIL`
- `EV_TIMEOUT`
- `EV_ESTOP`
- `EV_COMMS_LOST`
- `EV_RESET`

### Guard Conditions (Examples)
- `G1: retry_count < MAX_RETRIES`
- `G2: battery_ok == true`
- `G3: object_present == true`

### Timers (Typical)
- `T_SCAN = 5s`
- `T_DECODE = 3s`
- `T_ALIGN = 4s`
- `T_GRASP = 2s`
- `T_COMMS = 1s` heartbeat timeout

---

## 2) High-Level Flow

1. **IDLE → SCAN_QR → DECODE_QR → PLAN_MISSION**
2. **NAVIGATE_TO_PICK → ALIGN_FOR_PICK → ARM_PREPARE → GRASP → LIFT**
3. **NAVIGATE_TO_PLACE → ALIGN_FOR_PLACE → PLACE → RESET_ARM → DONE**

---

## 3) Transition Table

> Notes:
> - **Action** describes what is executed on transition.
> - **On Entry** actions are listed per state in Section 4.

| Current State | Event / Condition | Guard | Action (on transition) | Next State |
|---|---|---|---|---|
| IDLE | EV_START | G2 | init_modules(); start_camera(); retry_count=0 | SCAN_QR |
| IDLE | EV_ESTOP | — | safe_stop_all(); | ERROR |
| SCAN_QR | EV_QR_DETECTED | — | freeze_base(); capture_frame(); start_timer(T_DECODE) | DECODE_QR |
| SCAN_QR | EV_TIMEOUT (T_SCAN) | G1 | retry_count++; re_scan(); start_timer(T_SCAN) | SCAN_QR |
| SCAN_QR | EV_TIMEOUT (T_SCAN) | !G1 | log("QR not found"); | ERROR |
| SCAN_QR | EV_COMMS_LOST | — | safe_stop_base(); | ERROR |
| DECODE_QR | EV_QR_DECODED_OK | — | validate_payload(); stop_timer(T_DECODE) | PLAN_MISSION |
| DECODE_QR | EV_QR_DECODED_FAIL | G1 | retry_count++; clear_frame(); start_timer(T_SCAN) | SCAN_QR |
| DECODE_QR | EV_TIMEOUT (T_DECODE) | G1 | retry_count++; start_timer(T_SCAN) | SCAN_QR |
| DECODE_QR | EV_TIMEOUT (T_DECODE) | !G1 | log("Decode failed"); | ERROR |
| PLAN_MISSION | payload_valid == true | — | target = map_payload_to_zone(); plan_path(); | NAVIGATE_TO_PICK |
| PLAN_MISSION | payload_valid == false | — | log("Invalid payload"); | ERROR |
| NAVIGATE_TO_PICK | EV_OBSTACLE | — | obstacle_avoid(); | NAVIGATE_TO_PICK |
| NAVIGATE_TO_PICK | EV_TARGET_REACHED | — | stop_base(); start_timer(T_ALIGN) | ALIGN_FOR_PICK |
| NAVIGATE_TO_PICK | EV_TIMEOUT | — | log("Nav timeout"); | ERROR |
| ALIGN_FOR_PICK | EV_ALIGNED_OK | — | lock_base(); stop_timer(T_ALIGN) | ARM_PREPARE |
| ALIGN_FOR_PICK | EV_TIMEOUT (T_ALIGN) | G1 | retry_count++; refine_alignment(); start_timer(T_ALIGN) | ALIGN_FOR_PICK |
| ALIGN_FOR_PICK | EV_TIMEOUT (T_ALIGN) | !G1 | log("Align failed"); | ERROR |
| ARM_PREPARE | arm_ready == true | — | move_to_pregrasp(); | GRASP |
| ARM_PREPARE | EV_TIMEOUT | — | log("Arm prepare fail"); | ERROR |
| GRASP | EV_GRIP_OK | — | confirm_grip(); start_timer(1s) | LIFT |
| GRASP | EV_GRIP_FAIL | G1 | retry_count++; open_gripper(); micro_adjust(); | GRASP |
| GRASP | EV_GRIP_FAIL | !G1 | log("Grip failed"); | ERROR |
| LIFT | lift_done == true | — | move_arm_transport_pose(); | NAVIGATE_TO_PLACE |
| LIFT | EV_TIMEOUT | — | log("Lift fail"); | ERROR |
| NAVIGATE_TO_PLACE | EV_OBSTACLE | — | obstacle_avoid(); | NAVIGATE_TO_PLACE |
| NAVIGATE_TO_PLACE | EV_TARGET_REACHED | — | stop_base(); start_timer(T_ALIGN) | ALIGN_FOR_PLACE |
| ALIGN_FOR_PLACE | EV_ALIGNED_OK | — | lock_base(); stop_timer(T_ALIGN) | PLACE |
| ALIGN_FOR_PLACE | EV_TIMEOUT (T_ALIGN) | G1 | retry_count++; refine_alignment(); start_timer(T_ALIGN) | ALIGN_FOR_PLACE |
| ALIGN_FOR_PLACE | EV_TIMEOUT (T_ALIGN) | !G1 | log("Place align failed"); | ERROR |
| PLACE | place_done == true | — | open_gripper(); | RESET_ARM |
| PLACE | EV_TIMEOUT | — | log("Place fail"); | ERROR |
| RESET_ARM | arm_home == true | — | unlock_base(); retry_count=0; | DONE |
| DONE | EV_START | — | start_camera(); | SCAN_QR |
| ANY | EV_ESTOP | — | safe_stop_all(); | ERROR |
| ANY | EV_COMMS_LOST | — | safe_stop_all(); | ERROR |
| ERROR | EV_RESET | — | clear_faults(); reinit_modules(); | IDLE |

---

## 4) Entry / Exit Actions

### Entry Actions
- **SCAN_QR (entry)**: `start_timer(T_SCAN); camera_on();`
- **DECODE_QR (entry)**: `decode_frame();`
- **NAVIGATE_* (entry)**: `base_set_speed(profile); enable_obstacle_monitor();`
- **ALIGN_* (entry)**: `base_set_speed(slow); enable_alignment_logic();`
- **GRASP (entry)**: `gripper_open(); approach_object(); gripper_close();`
- **ERROR (entry)**: `safe_stop_all(); signal_fault_led();`

### Exit Actions (examples)
- Leaving **ALIGN**: `disable_alignment_logic();`
- Leaving **NAVIGATE**: `stop_base();`

---

## 5) Retry Policy

- `MAX_RETRIES = 3` (recommended)
- Retries apply to:
  - QR scan/decode
  - Alignment
  - Grasping
- On exceeding retries → transition to **ERROR**.

---

## 6) Safety & Fault Handling

### Immediate ERROR triggers
- Emergency stop pressed
- Communication lost (heartbeat timeout)
- Motor stall / overcurrent (if monitored)
- Critical battery low

### Safe Stop Behavior
- Stop base motors
- Disable arm motion
- Open gripper (optional, depending on safety)
- Log error reason

---

## 7) Implementation Notes

- Use an **event queue** (FIFO) for sensor and vision events.
- Keep the state machine loop non-blocking.
- Use timers instead of `delay()` where possible.
- Log every transition:
  - `timestamp, state_from, event, state_to`
