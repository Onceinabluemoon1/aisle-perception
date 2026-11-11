# RGB-D Replay-Based Stopping in Aisles
Implement stopping behavior based on replayed, synthetically generated four-segment RGB-D data. 
Can replace by real world `.mcap` files for improvement for next version. 
The input consists of pre-recorded sequences rather than data from an actual vehicle or its onboard sensors.
When a person enters the stop zone within 2.5 meters in front of the vehicle, the system issues a 
`STOP_PERSON` signal across two consecutive perception frames, commanding a speed of 0. For objects like totes or pallets 
located within the aisle, the system calculates the distance in meters. If an object is within 1.0 meter, it issues a 
`STOP_DISTANCE` signal. These distances are derived from depth back-projection. Even if the process responsible for
calculating speed commands is disabled, the system continues to output `STOP_PERSON` when a person is detected. 
The safety gate logic does not rely on the output of the speed-command process.
## Data Source
Visual frames, depth data, and annotations are generated using `aisle_replay/synth.py`.
- The camera follows a pinhole model, with its optical axis tilted 27° downward from the horizontal plane. The `base_link` frame is defined with x pointing forward, y to the left, and z upward. The camera is positioned at `(0.20, 0.00, 0.55)` meters.
- Depth values ​​represent the distance along the camera's optical axis z-axis, stored as `uint16` in millimeters, and are subsequently back-projected into the `base_link` coordinate frame.
- People, totes, and pallets are represented as solid-colored rectangular cuboids. Other "distractor" objects are not labeled.
The YOLO11-seg model is fine-tuned for 40 epochs using Ultralytics' pre-trained `yolo11n-seg` weights ([v8.3.0 assets](https://github.com/ultralytics/assets/releases/download/v8.3.0/yolo11n-seg.pt)), trained on 64 synthetic images and validated on 16 synthetic images. There are three mutually independent categories: `person`, `tote`, and `pallet`. The `person` category is not grouped into `OTHER`. The fine-tuned weights are located at `aisle_replay/weights/yolo11n-seg-synth.pt`.
## Four Replay Segments
| Segment | Expected Outcome |
| --- | --- |
| `empty` | `OK` for every frame; speed is 0.6 m/s as per the rules. |
| `person` | `OK` when the person is beyond 2.5 m; `STOP_PERSON` triggered from the frame where the person's front surface reaches 2.2 m. |
| `tote` | `STOP_DISTANCE` when inside the lane with the front surface at 0.75 m; `OK` if the tote is outside the lane. |
| `pallet` | `STOP_DISTANCE` when inside the lane with the front surface at 0.80 m; `OK` if the pallet is outside the lane. |
`tote_1m` is not a behavioral segment; the front surface of the tote is fixed at 1.00 m, used for measuring distance error.
Thresholds are defined in `aisle_replay/safety/rules.yaml`: 2.5 m for a person, 1.0 m for a tote or pallet, and a lane half-width of 0.90 m.
