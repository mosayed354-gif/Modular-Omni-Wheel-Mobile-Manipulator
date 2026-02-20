stateDiagram-v2
    [*] --> IDLE

    IDLE --> SCAN_QR: EV_START
    IDLE --> ERROR: EV_ESTOP

    SCAN_QR --> DECODE_QR: EV_QR_DETECTED
    SCAN_QR --> SCAN_QR: T_SCAN timeout / retry++
    SCAN_QR --> ERROR: T_SCAN timeout & retry>=MAX
    SCAN_QR --> ERROR: EV_COMMS_LOST

    DECODE_QR --> PLAN_MISSION: EV_QR_DECODED_OK
    DECODE_QR --> SCAN_QR: EV_QR_DECODED_FAIL / retry++
    DECODE_QR --> SCAN_QR: T_DECODE timeout / retry++
    DECODE_QR --> ERROR: T_DECODE timeout & retry>=MAX

    PLAN_MISSION --> NAVIGATE_TO_PICK: payload_valid
    PLAN_MISSION --> ERROR: !payload_valid

    NAVIGATE_TO_PICK --> NAVIGATE_TO_PICK: EV_OBSTACLE / avoid()
    NAVIGATE_TO_PICK --> ALIGN_FOR_PICK: EV_TARGET_REACHED
    NAVIGATE_TO_PICK --> ERROR: NAV timeout

    ALIGN_FOR_PICK --> ARM_PREPARE: EV_ALIGNED_OK
    ALIGN_FOR_PICK --> ALIGN_FOR_PICK: T_ALIGN timeout / retry++
    ALIGN_FOR_PICK --> ERROR: T_ALIGN timeout & retry>=MAX

    ARM_PREPARE --> GRASP: arm_ready
    ARM_PREPARE --> ERROR: arm timeout

    GRASP --> LIFT: EV_GRIP_OK
    GRASP --> GRASP: EV_GRIP_FAIL / retry++
    GRASP --> ERROR: EV_GRIP_FAIL & retry>=MAX

    LIFT --> NAVIGATE_TO_PLACE: lift_done
    LIFT --> ERROR: lift timeout

    NAVIGATE_TO_PLACE --> NAVIGATE_TO_PLACE: EV_OBSTACLE / avoid()
    NAVIGATE_TO_PLACE --> ALIGN_FOR_PLACE: EV_TARGET_REACHED

    ALIGN_FOR_PLACE --> PLACE: EV_ALIGNED_OK
    ALIGN_FOR_PLACE --> ALIGN_FOR_PLACE: T_ALIGN timeout / retry++
    ALIGN_FOR_PLACE --> ERROR: T_ALIGN timeout & retry>=MAX

    PLACE --> RESET_ARM: place_done
    PLACE --> ERROR: place timeout

    RESET_ARM --> DONE: arm_home
    DONE --> SCAN_QR: EV_START

    state ANY <<choice>>
    ERROR --> IDLE: EV_RESET
