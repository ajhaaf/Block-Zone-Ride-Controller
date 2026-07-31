# Block-Zone-Ride-Controller
Real-Time Systems Final Capstone

Theme: I aim to build an autonomous home roller coaster to act as a living display item in my office. This project begins that work by building a real-time autonomous block-zone and dispatch controller, capable of engaging brakes if an e-stop is triggered or if more than one vehicle is detected within the block zone.

Video: https://youtu.be/F9B1JHFVAPU
Wokwi: https://wokwi.com/projects/470820577391242241

Hardware Architecture:
Component      |Function/Parallel                  |Hardware Pin      |Protocol/Driver
ESP32          |Master Controller                  |                  |FreeRTOS
E-Stop Button  |Hardware E-Stop Active-Low         |GPIO18            |Hardware Edge Interrupt
Dispatch Button|Station Operator Control Active-Low|GPIO19            |Hardware Edge Interrupt
Servo          |Station pneumatic brake fin        |GPIO13            |ESP-IDF Native PWM

Software Architecture:
Core 1: Deterministic Task Execution, high-priority ISRs, queue processing, safety verification
Core 0: Wi-Fi networking, HTTP web console rendering
[ CORE 1: REAL-TIME SAFETY PLANE ]                  [ CORE 0: OBSERVABILITY ]
                                                                                   
  +--------------------+      20Hz Telemetry      +--------------------+          +-----------------------+
  |   producer_task    | ─── (Queue: 16 Frames) ──>|   consumer_task    |          |  serial_monitor_task  |
  |  (Priority 8)      |                          |  (Priority 8)      |          |  (Priority 4)         |
  +--------------------+                          +--------------------+          +-----------+-----------|
            │                                               │                                 │
            └───────────────> [ Event Group ] <─────────────┘                                 │ Reads Atomic
                                      │                                                       │ Heartbeats
                                      ▼                                                       │ & Queue Depth
                          +-----------------------+                                           │
                          |   coordinator_task    | <── Direct Task Notification ─────────────┘
                          |  (Priority 9)         |     (Dispatch Button - GPIO 19)
                          +-----------------------+
                                      │
                                      ▼ (Triggers Dispatch Sequence / Servo to 90°)
                                      
  +--------------------+
  |  button_estop_isr  | ── Direct Task Notification ──> +--------------------+
  |  (GPIO 18 Interrupt)                                 |   responder_task   | ──> [ LEDC Servo @ GPIO 13 ]
  +--------------------+                                 |  (Priority 12)     |     (E-Stop: Instant 0°)
                                                         +--------------------+

Task Scheduling & Rate-Monotonic Analysis

Task Name       |Priority     |Core|Period (T)   |WCET (C)|Description & Safety Mechanism
button_estop_isr|Hardware ISR |1    |Unscheduled |0.05 ms |Immediate preemption; gives direct task notification to responder_task.
responder_task  |High (12)    |1    |Event-Driven|0.15 ms |Hard Real-Time E-Stop; sets LEDC PWM to 0∘ (Engage Brakes).
coordinator_task|Med-High (9) |1    |50 ms       |0.80 ms|Evaluates block occupancy & E-Stop state before authorizing vehicle dispatch.
producer_task   |Medium (8)   |1    |50 ms (20Hz)|0.35 ms|Generates track speed (mm/s) & zone ID telemetry packets.
consumer_task   |Medium (8)   |1    |50 ms       |1.10 ms|Performs overspeed checks (>3.2 m/s) & sets fault bits in Event Group.
monitor_task    |Low (4)      |0    |1000 ms     |12.0 ms|Formats real-time metrics for Serial / Web dashboard on Core 0.

Hazard Analysis & Industry Safety Mapping (ASTM F2291 / ISO 13849)

Hazard ID|Fail Mode	|Risk Level |Mitigation Strategy|Standard Mapping
HAZ-01   |Operator launches vehicle while down-track block is occupied|CATASTROPIC|coordinator_task evaluates last_consumed_frame.block_zone_id before releasing brakes. Rejects launch if Zone 2 is occupied.|ASTM F2291 (Zone Interlocking)

HAZ-02|Vehicle overspeed on circuit|HIGH|"consumer_task detects velocity >3.2 m/s| sets EV_BIT_FAULT_INJECTED| and logs a fault event."|ISO 13849-1 PL-d (Safety Function)

HAZ-03|Immediate emergency in station bay|HIGH|Hardware interrupt on GPIO 18 bypasses IPC queues and preempts lower priority tasks to drop brake fins (0∘) within microseconds.|ASTM F2291 (E-Stop Response)

HAZ-04|Telemetry queue saturation/buffer overflow|MEDIUM|Queue sized to 16 frames (800 ms burst buffer). Back-pressure policy drops non-critical telemetry frames without stalling safety tasks.|IEC 61508 (Software Architecture)

