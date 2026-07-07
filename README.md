# Coverage Planner

ROS 2 Python package for generating autonomous UAV coverage paths and uploading
missions to a MAVLink-compatible flight controller. The package focuses on
lawnmower/boustrophedon waypoint planning for aerial coverage, mission upload
through `pymavlink`, and lightweight ROS 2 publication of CubeOrange GPS, IMU,
and odometry data.

## What This Repository Contains

- Coverage waypoint generators for square, circular, polygon, and science-style
  waypoint patterns.
- MAVLink mission upload scripts that add takeoff, waypoint traversal, and
  landing mission items.
- Utility scripts for arming, disarming, landing, takeoff/land testing, and
  CubeOrange connectivity checks.
- ROS 2 publishers for `/odom_cube`, `/gps_cube`, and `/imu_cube`.
- Sample mission and boundary JSON files under `src/Coverage_Planner/Waypoints`
  and `src/Coverage_Planner/Coverage_Planner/CoverageBoundary`.

## Repository Layout

```text
.
├── README.md
├── coverage_boundary.json
└── src/
    └── Coverage_Planner/
        ├── Coverage_Planner/
        │   ├── CoverageWP.py             # Polygon boundary coverage generator
        │   ├── CircularCoverage.py       # Circular coverage generator
        │   ├── SquareCoverage.py         # Square coverage generator
        │   ├── ScienceWaypointGen.py     # Multi-point science waypoint generator
        │   ├── Mission.py                # Coverage mission upload/execution
        │   ├── ScienceMission.py         # Science mission upload/execution
        │   ├── OdometryPub.py            # CubeOrange odometry/GPS/IMU publisher
        │   ├── Waypoint.py               # Manual local waypoint capture workflow
        │   ├── ArmDisarm.py              # Arm/disarm utility
        │   ├── Land.py                   # Land command utility
        │   └── CoverageBoundary/         # Saved boundary files
        ├── Waypoints/                    # Sample/generated waypoint JSON files
        ├── package.xml
        ├── setup.py
        └── test/
```

## Core Concepts

The waypoint generators convert GPS coordinates into UTM coordinates, create a
coverage pattern in meters, and convert generated points back to latitude and
longitude. The main generated output is a JSON object containing a
`lap_waypoints` array in the given format:

```json
{
  "lap_waypoints": [
    {
      "latitude": -35.3614651,
      "longitude": 149.1652373,
      "altitude": 20
    }
  ]
}
```

Mission scripts read this waypoint data, convert latitude and longitude into
MAVLink `lat * 1e7` / `lon * 1e7` format, upload mission items to the flight
controller, arm the vehicle, take off, switch to AUTO, and monitor progress.

## Requirements

This package is structured as an `ament_python` ROS 2 package.

ROS dependencies declared in `package.xml`:

- `rclpy`
- `pymavlink`
- `sensor_msgs`
- `nav_msgs`
- `geometry_msgs`
- `tf_transformations`

Python libraries used by the scripts:

- `numpy`
- `matplotlib`
- `pyproj`
- `shapely`
- `scipy`
- `requests`

Install missing Python dependencies in the same environment used by ROS 2:

```bash
pip install numpy matplotlib pyproj shapely scipy requests pymavlink
```

## Build

From the repository root:

```bash
colcon build --packages-select Coverage_Planner
source install/setup.bash
```

If you are using `zsh`, source the generated `zsh` setup file instead:

```bash
source install/setup.zsh
```

## Running Waypoint Generators

Generate square coverage waypoints:

```bash
ros2 run Coverage_Planner SquareWPGeneration
```

Generate circular coverage waypoints:

```bash
ros2 run Coverage_Planner CircleWPGeneration
```

Generate polygon-boundary coverage waypoints from a JSON coordinates file:

```bash
ros2 run Coverage_Planner CoverageGeneration --ros-args \
  -p coords_file:=/absolute/path/to/boundary.json \
  -p fov_x_deg:=[CAMERA_X_FOV] \
  -p fov_y_deg:=[CAMERA_Y_FOV] \
  -p altitude_feet:=[UAV_ALTITUDE
```bash
ros2 run Coverage_Planner ScienceWPGeneration
```

Most generators save output into the installed package share directory:

```text
install/Coverage_Planner/share/Coverage_Planner/Waypoints/
```

Several scripts also display a `matplotlib` plot of the planned path.

## Running Mission Utilities

Run the main coverage mission workflow:

```bash
ros2 run Coverage_Planner CoverageMission
```

Publish CubeOrange odometry, GPS, and IMU data:

```bash
ros2 run Coverage_Planner Odometry
```

Run arm/disarm test against SITL/UDP:

```bash
ros2 run Coverage_Planner ArmCheck
```

Send a land command against SITL/UDP:

```bash
ros2 run Coverage_Planner Land
```

Other console scripts declared in `setup.py` include:

- `Takeoff_Land`
- `WaypointManual`
- `CubeOrangeTest`
- `ScienceMission`
- `Disarm`

Note: `setup.py` currently declares `Connect = Coverage_Planner.Connect:main`,
but there is no `Connect.py` file in the package. That entry point will fail
until the module is added or the entry is removed.

## MAVLink Connection Assumptions

Connection strings are currently hard-coded in the scripts:

- CubeOrange Serial Connection uses `/dev/ttyACM0`. The correct Serial Port can be checked using the following command:

```bash
ls /dev | grep tty
```

- `ScienceMission.py` contains hard-coded HTTP endpoints for waypoint
  start/stop notifications, to collect and plot Sensor.

Before running against real hardware, inspect the target script and update the
connection string, waypoint file, altitude, and mission parameters for your
vehicle and test environment.

## Safety Notes

This repository contains scripts that can arm, take off, upload missions, and
command landing through MAVLink. Test in SITL before using physical hardware.
Use a safety pilot, verify geofence/failsafe settings, confirm the mission file,
and ensure the flight controller mode mapping matches the numeric mode IDs used
in the scripts.

Several mission workflows include a monitor thread intended to return control to
the safety pilot when POSHOLD is detected and throttle is above a threshold.
Treat this as an extra guard, not as the only safety mechanism.

## Development Notes

- The package name is `Coverage_Planner`, including capitalization.
- Generated waypoint JSON is expected to contain `lap_waypoints`. The coverage boundary waypoints are generated from the [MissionBuilder](https://github.com/MrigaankMouli/Mission-Builder.git) repository
- This repo uses use UTM zone `18` directly. Update the projection logic if your
  operating area is outside that zone.
