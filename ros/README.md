# `patchworkpp` ROS 2 package

## Table of contents

- [Overview](#overview)
- [Installation & Build](#installation--build)
- [Nodes / Executables](#nodes--executables)
- [Launch Files](#launch-files)
- [Interfaces](#interfaces)
- [Configuration](#configuration)
- [Testing](#testing)
- [Known Issues](#known-issues)
- [Glossary](#glossary)

## Overview

`patchworkpp` is the ROS 2 wrapper for the bundled Patchwork++ C++ ground-segmentation algorithm. It accepts an XYZ `PointCloud2`, partitions the points with Patchwork++'s concentric-zone model, and publishes the reconstructed input plus ground and non-ground point clouds. In a larger perception stack it is a LiDAR preprocessing component: downstream obstacle detection or mapping can consume the non-ground result while ground-aware consumers use the ground result. The surrounding system and its downstream consumers are not present here, so that system-level role is inferred from topic names and the algorithm [UNCLEAR — verify].

- **Package version:** `0.1.0` in `package.xml`; the CMake project/core library reports `1.0.4` (`package.xml:4`, `CMakeLists.txt:2`, `../cpp/CMakeLists.txt:2`).
- **Build type:** `ament_cmake` (`package.xml:9`, `package.xml:24`).
- **Language/standard:** C++, compiled as C++20 (`CMakeLists.txt:34`).
- **ROS distribution:** CI builds both ROS 2 Humble and Jazzy (`../.github/workflows/ros.yml:12-15`). Humble is therefore a supported assumption; no distro-specific compile-time branch exists.
- **License metadata:** `package.xml` says GPL-3.0, while `LICENSE` in this package is MIT and the repository-root license is BSD-2-Clause. See [Known Issues](#known-issues).

### Package dependencies

`<depend>` means a build, export, and runtime dependency in package format 3. No listed ROS dependency is another package in this repository/workspace; all are external.

| Name | Manifest type | Internal? | Purpose observed in this package |
| --- | --- | --- | --- |
| `ament_cmake` | `buildtool_depend` | No | Configures, installs, and exports the CMake package. |
| `rcutils` | `depend` | No | Declared and linked through `ament_target_dependencies`; no direct API use was found. |
| `rclcpp` | `depend` | No | Node, parameters, publishers, subscription, logging, and QoS. |
| `rclcpp_components` | `depend` | No | Component registration and generated standalone executable. |
| `geometry_msgs` | `depend` | No | Two headers are included but their types are not used at runtime (`src/GroundSegmentationServer.cpp:8-9`). |
| `nav_msgs` | `depend` | No | `Odometry` is included but unused (`src/GroundSegmentationServer.cpp:10`). |
| `std_msgs` | `depend` | No | Uses `std_msgs/msg/Header`, via the `PointCloud2` conversion helpers; `String` is included but unused. |
| `sensor_msgs` | `depend` | No | Input and output `sensor_msgs/msg/PointCloud2` messages and iterators. |
| `tf2` | `depend` | No | Declared but no direct use was found. |
| `tf2_ros` | `depend` | No | Found and linked, but no transform buffer/listener/broadcaster is created. |
| `ros2launch` | `exec_depend` | No | Runs the installed Python launch file. |

Additional effective dependencies are not fully represented by `package.xml`:

| Name | Kind | Use |
| --- | --- | --- |
| Eigen 3.4.0 or system Eigen3 | C++ build dependency | Linear algebra in the core algorithm. By default CMake downloads Eigen 3.4.0 and applies `../cpp/cmake/eigen.patch`; `-DUSE_SYSTEM_EIGEN3=ON` uses an installed Eigen package. |
| `rviz2` | Optional runtime dependency | Started when `visualize` is true. Not declared in `package.xml`. |
| `ros2 bag` / `ros2bag` | Optional runtime dependency | Started when `bagfile` is non-empty. Not declared in `package.xml`. Storage plugins required by a particular bag format are also external. |
| `patch` and network access | Default build-time tools | Required by the default Eigen `FetchContent` path (`../cpp/cmake/eigen.cmake:30-32`). |

## Installation & Build

Place the full repository in a colcon workspace: the ROS package is the `ros/` directory, and its CMake expects the sibling `../cpp/` directory. From the workspace root:

```bash
source /opt/ros/humble/setup.bash
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select patchworkpp
source install/setup.bash
```

The default build downloads Eigen 3.4.0. To avoid a network fetch when a system Eigen3 development package is installed:

```bash
colcon build --packages-select patchworkpp \
  --cmake-args -DUSE_SYSTEM_EIGEN3=ON
```

This second form was verified in this checkout on ROS 2 Humble. CMake unconditionally sets `CMAKE_BUILD_TYPE=Release` and exports `compile_commands.json` (`CMakeLists.txt:5-6`); a caller-supplied build type is consequently overridden. The ROS wrapper and core both use C++20.

### Non-standard build and generated artifacts

The package adds `../cpp/` as a subdirectory and builds the bundled `patchworkpp::ground_seg_cores` static library. The alternative branch for a checkout without `../cpp/` only prints a fetch message; it does not declare or populate Patchwork++ and is not a working out-of-tree build (`CMakeLists.txt:12-22`). There is no `rosidl` generation.

Installed/exported artifacts are:

| Artifact | Install destination | Source |
| --- | --- | --- |
| `patchworkpp_node` executable | `lib/patchworkpp/` | Generated by `rclcpp_components_register_node` from the component registration (`CMakeLists.txt:47`). |
| `libgseg_component.so` | `lib/` | `src/GroundSegmentationServer.cpp`. |
| `libground_seg_cores.a` | `lib/` | `../cpp/patchworkpp/src/patchworkpp.cpp`. |
| Core public headers | `include/patchwork/` | `../cpp/patchworkpp/include/patchwork/`. |
| Core CMake exports | `lib/cmake/patchworkpp/` | `patchworkpp::ground_seg_cores` config/targets (`../cpp/patchworkpp/CMakeLists.txt:20-42`). |
| Component resource entry | ament resource index | Plugin `patchworkpp_ros::GroundSegmentationServer`. |
| Launch files | `share/patchworkpp/launch/` | `launch/`. |
| RViz files | `share/patchworkpp/rviz/` | `rviz/`. |

The repository's Open3D scripts and C++/Python demo targets are not part of the normal ROS package build: `INCLUDE_CPP_EXAMPLES` defaults to `OFF`, and the Python tree has `COLCON_IGNORE`.

## Nodes / Executables

### `patchworkpp_node`

| Property | Value |
| --- | --- |
| Executable | `patchworkpp_node` |
| Component plugin | `patchworkpp_ros::GroundSegmentationServer` |
| Source | `src/GroundSegmentationServer.cpp`, declaration in `src/GroundSegmentationServer.hpp`, conversions in `src/Utils.hpp` |
| Node name | `patchworkpp_node` (`src/GroundSegmentationServer.cpp:25-26`) |
| Default namespace | Root namespace `/`; the source does not set a namespace. |
| Lifecycle | Ordinary `rclcpp::Node`, not a lifecycle node. |
| Composable | Yes. The shared-library plugin is registered in both C++ and CMake (`src/GroundSegmentationServer.cpp:102-103`, `CMakeLists.txt:47`). |
| Standalone executor | `SingleThreadedExecutor`, the default selected by `rclcpp_components_register_node` because no `EXECUTOR` argument is supplied. |
| Callback groups | None created explicitly; the subscription uses the node's default mutually-exclusive callback group. A component container may choose a different executor, but this node still has one subscription callback. |

Run it directly:

```bash
ros2 run patchworkpp patchworkpp_node
```

Or load the component into an existing component manager:

```bash
ros2 component load /ComponentManager patchworkpp \
  patchworkpp_ros::GroundSegmentationServer
```

#### Published topics

All output messages contain only `x`, `y`, and `z` fields as `float32`; other input fields such as intensity, ring, and timestamps are discarded (`src/Utils.hpp:84-107`, `src/Utils.hpp:193-197`). Publishing is input-driven: one message is published on each output topic for every completed input callback, so the output rate is the processed input rate rather than a timer rate.

| Topic | Type | QoS | Behavior |
| --- | --- | --- | --- |
| `/patchworkpp/cloud` | `sensor_msgs/msg/PointCloud2` | Explicitly **reliable** and **transient local**. History and depth remain those of `rmw_qos_profile_default`, so their exact values are middleware-dependent [UNCLEAR — verify]. | Reconstructed XYZ-only copy of the input. It preserves the input stamp and frame ID, except a leading `/` is removed. |
| `/patchworkpp/ground` | `sensor_msgs/msg/PointCloud2` | Same as above. | Points classified as ground. The input stamp is preserved and `frame_id` is replaced by `base_frame`. |
| `/patchworkpp/nonground` | `sensor_msgs/msg/PointCloud2` | Same as above. | Points classified as non-ground, including points outside `(min_range, max_range]`. The input stamp is preserved and `frame_id` is replaced by `base_frame`. |

The output names begin with `/`, so they are absolute and are not placed under the node namespace (`src/GroundSegmentationServer.cpp:64-71`).

#### Subscribed topics

| Topic | Type | QoS | Callback behavior |
| --- | --- | --- | --- |
| `pointcloud_topic` | `sensor_msgs/msg/PointCloud2` | `rclcpp::SensorDataQoS`: keep-last history, depth 5, best-effort reliability, volatile durability. | Reads `x`, `y`, and `z` as `float32` into an Eigen matrix; runs Patchwork++ synchronously; then publishes all three outputs (`src/GroundSegmentationServer.cpp:53-56`, `src/GroundSegmentationServer.cpp:76-88`). The relative name resolves under the node namespace and is remapped by the supplied launch file. |

The conversion assumes the incoming cloud contains `float32` fields named exactly `x`, `y`, and `z`. There is no validation or exception handling around missing/wrongly typed fields (`src/Utils.hpp:161-174`).

#### Services and actions

The source creates no service servers/clients and no action servers/clients. Standard services provided internally by `rclcpp::Node` (for example parameter services when enabled by `NodeOptions`) are not package-specific APIs.

#### Parameters

No parameter descriptors are supplied. Consequently none is descriptor-level read-only. Each declared value is copied into the Patchwork++ instance during construction, and there is no parameter callback; changing it later through ROS parameter services does **not** reconfigure the running algorithm. The “launch value” column is the effective override requested by `patchworkpp.launch.py`, not the source default.

| Name | Type | Source default | Launch value | Meaning / runtime mutability |
| --- | --- | ---: | ---: | --- |
| `base_frame` | string | `base_link` | `velodyne` | Frame ID written onto ground/non-ground outputs. Startup-only in practice. |
| `sensor_height` | double | `1.723` | `0.8` | Sensor height used for initial seed selection and adaptive elevation. Startup-only. |
| `num_iter` | integer | `3` | `3` | PCA/plane-fitting iterations per patch. Startup-only. |
| `num_lpr` | integer | `20` | `10` | Maximum lowest-point representatives used to compute the initial height. Startup-only. |
| `num_min_pts` | integer | `10` | `0` | Patches with fewer points are sent directly to non-ground. Startup-only. See Known Issues for the launch value. |
| `th_seeds` | double | `0.125` | `0.3` | Ground initial-seed height threshold. Startup-only. |
| `th_dist` | double | `0.125` | `0.125` | Ground plane thickness/distance threshold. Startup-only. |
| `th_seeds_v` | double | `0.25` | `0.25` | Vertical-plane initial-seed threshold. Startup-only. |
| `th_dist_v` | double | `0.1` | `0.9` | Vertical-structure distance threshold. Startup-only. |
| `max_range` | double | `80.0` | `50.0` | Inclusive outer segmentation range in metres; farther points become non-ground. Startup-only. |
| `min_range` | double | `2.7` | `1.0` | Exclusive inner segmentation range in metres; nearer points become non-ground. Startup-only. |
| `uprightness_thr` | double | `0.707` | `0.101` | Minimum plane-normal Z component used by GLE. Startup-only. |
| `verbose` | bool | Algorithm default `false` | `true` (requested, but ignored) | Read with `get_parameter(..., false)` but never declared (`src/GroundSegmentationServer.cpp:44`). Default `NodeOptions` do not automatically declare overrides, so the supplied launch value is not set and the fallback remains `false`. |
| `use_sim_time` | bool | rclcpp default `false` | `false` | Standard ROS clock parameter supplied by launch; unlike the copied algorithm settings, rclcpp handles changes to this parameter. It is not explicitly declared by this source. |

The core also has fixed, non-ROS-configurable settings: RNR is forcibly disabled by the wrapper, while R-VPF and TGR retain core defaults of enabled; zone/ring counts, adaptive thresholds, and storage sizes retain defaults from `../cpp/patchworkpp/include/patchwork/patchworkpp.h:88-127`.

## Launch Files

### `launch/patchworkpp.launch.py`

Starts `patchworkpp_node`, optionally starts RViz with the installed configuration, and optionally invokes `ros2 bag play`. The file uses `LaunchConfiguration(..., default=...)` directly and declares no `DeclareLaunchArgument` actions. Command-line overrides are used as launch configurations, but descriptions are not registered; on Humble, `ros2 launch ... --show-args` reports “No arguments.” Other distributions were not checked [UNCLEAR — verify].

| Launch configuration | Default | Description / use |
| --- | --- | --- |
| `use_sim_time` | `false` | Passed to the node as the standard ROS time parameter. |
| `base_frame` | `velodyne` | Passed as `base_frame`. |
| `cloud_topic` | `/velodyne_points` | Remaps the node's `pointcloud_topic` subscription. |
| `visualize` | `false` | If true, starts `rviz2 -d <share>/rviz/patchworkpp.rviz`. |
| `bagfile` | empty string | If non-empty, runs `ros2 bag play <bagfile>` once; no loop or storage option is added. |

The node also receives the fixed algorithm parameter mapping shown in the preceding table (`launch/patchworkpp.launch.py:41-60`). These algorithm settings cannot be overridden through this launch file's CLI because they are Python literals. The unused `config` class at `launch/patchworkpp.launch.py:13-16` does not affect the launch description.

Example:

```bash
ros2 launch patchworkpp patchworkpp.launch.py \
  cloud_topic:=/velodyne_points \
  base_frame:=velodyne \
  visualize:=true \
  use_sim_time:=false
```

## Interfaces

This package defines no `.msg`, `.srv`, or `.action` files and performs no `rosidl` interface generation.

Runtime data exchange consumes and publishes `sensor_msgs/msg/PointCloud2`; it also uses the embedded `std_msgs/msg/Header`. The input fields actually consumed are:

| Field | Expected type | Use |
| --- | --- | --- |
| `x` | `float32` | Cartesian X coordinate. |
| `y` | `float32` | Cartesian Y coordinate. |
| `z` | `float32` | Cartesian Z coordinate. |

Helpers exist for timestamp fields named `t`, `timestamp`, or `time` with `uint32`, `float32`, or `float64` types (`src/Utils.hpp:27-81`), but the node callback does not call those helpers. Geometry and navigation message headers included by `GroundSegmentationServer.cpp` are unused and do not form runtime interfaces.

## Configuration

There is no `config/` directory and no ROS parameter YAML file. The launch file passes an inline Python parameter dictionary; direct execution uses the C++ defaults.

`rviz/patchworkpp.rviz` is installed and loaded only when `visualize:=true`. Its relevant settings are:

| Setting | Value |
| --- | --- |
| Fixed frame | `lexus3/os_center_a_laser_data_frame` |
| Enabled clouds | `/patchworkpp/ground` (green) and `/patchworkpp/nonground` (red) |
| Disabled input display | `/lexus3/os_center/points` |
| Output-display requested QoS | Reliable, volatile, keep-last depth 5 |
| Input-display requested QoS | Best effort, volatile, keep-last depth 5 |
| Time synchronization source | `nonground` |

The RViz fixed frame and input topic are static and do not follow the launch arguments.

## Testing

There is no `test/` directory, no `BUILD_TESTING` block, no `ament_lint` dependency, and no unit, integration, or launch tests. The repository CI only performs `colcon build` on Humble and Jazzy (`../.github/workflows/ros.yml:9-23`).

Run any tests added in the future with:

```bash
colcon test --packages-select patchworkpp
colcon test-result --verbose
```

Notable uncovered behavior includes point-field validation/conversion, empty and sparse clouds, parameter override handling, QoS compatibility, component loading, launch conditionals, algorithm classification accuracy, frame semantics, and all error paths. An isolated Humble build was performed while preparing this document; it is a build check, not a test suite.

## Known Issues

### TODO / FIXME / HACK / XXX comments

| Location | Marker | Impact |
| --- | --- | --- |
| `src/GroundSegmentationServer.cpp:46` | `ToDo. Support intensity` | The wrapper forces reflected-noise removal off and strips intensity from all outputs. |
| `launch/patchworkpp.launch.py:14` | `TBU` | The example `config` class is incomplete and unused. |

No `FIXME`, `HACK`, or `XXX` comments were found in the ROS wrapper or bundled core used by it.

### Behavioral and packaging issues

- `verbose` is passed by launch but is not declared by the node. With the default `NodeOptions` used by the generated executable, overrides are not automatically declared, so `get_parameter(..., false)` returns the fallback and launch cannot enable verbose output (`src/GroundSegmentationServer.cpp:44`, `launch/patchworkpp.launch.py:59`). A custom component host could change `NodeOptions` [UNCLEAR — verify for that host].
- The launch file sets `num_min_pts` to `0`. Empty patches are therefore not rejected by the `< num_min_pts` guard, while later code reads plane state after `estimate_plane()` returns early on an empty set. The exact failure/classification effect depends on retained Eigen state [UNCLEAR — verify] (`../cpp/patchworkpp/src/patchworkpp.cpp:46-47`, `../cpp/patchworkpp/src/patchworkpp.cpp:191-221`).
- Ground/non-ground points are not transformed into `base_frame`; only the header string is replaced. If input coordinates are not already expressed in that frame, the output is mislabeled (`src/GroundSegmentationServer.cpp:90-98`). Despite declared `tf2`/`tf2_ros` dependencies, no transform is performed.
- Absolute output topic names ignore node namespaces. Remap them explicitly when multiple instances are needed.
- Output clouds discard every field except XYZ. RNR is disabled because intensity is unavailable to the core, and timestamp extraction/deskew helpers are dead code in this node.
- The RViz configuration is dataset-specific: its Lexus fixed frame/input topic do not match launch defaults `velodyne` and `/velodyne_points`. Ground/non-ground still display only if TF can resolve the hard-coded fixed frame.
- Publisher QoS uses transient-local durability for high-bandwidth point clouds, retaining a sample for late joiners; the source comment itself suggests changing QoS for real-time operation (`src/GroundSegmentationServer.cpp:58-67`). RViz requests volatile durability, which is compatible with a transient-local publisher. No definite package-internal QoS incompatibility was found; external producer/consumer QoS is unavailable [UNCLEAR — verify].
- The manifest declares several unused dependencies (`rcutils`, `geometry_msgs`, `nav_msgs`, `tf2`, `tf2_ros`) and omits optional launch-time dependencies (`rviz2`, `ros2bag`) plus the effective Eigen/system-tool build requirements.
- `geometry_msgs` is passed to `ament_target_dependencies` without a direct `find_package(geometry_msgs REQUIRED)`, and `std_msgs` is used without either a direct `find_package` or target dependency. The current build succeeds through transitive discovery, but this is fragile (`CMakeLists.txt:24-45`).
- A repository layout without sibling `../cpp/` is announced as supported via fetching, but the branch is unimplemented and cannot provide `patchworkpp::ground_seg_cores` (`CMakeLists.txt:12-22`).
- `CMAKE_BUILD_TYPE` is forced to Release, overriding caller intent (`CMakeLists.txt:5`).
- Version metadata disagrees (`package.xml` 0.1.0 versus CMake/core 1.0.4), and license metadata disagrees (`package.xml` GPL-3.0, package `LICENSE` MIT, repository `LICENSE` BSD-2-Clause). Resolve these before redistribution.
- Algorithm timing is read into a local variable and then unused by the ROS node (`src/GroundSegmentationServer.cpp:86`). No diagnostic topic or metric is exposed.
- Algorithm parameters have no range validation and cannot be applied dynamically after construction.

### Documentation limits / missing context

- No upstream LiDAR publisher or downstream consumer is included, so their actual topic remaps, coordinate-frame contract, rates, and QoS requirements could not be verified.
- No bag metadata is included in the ROS package. Repository `.bin` samples belong to native demos and do not establish the deployed ROS point layout.
- The standalone component main is generated by `rclcpp_components` at build time rather than stored in this repository; its Humble default executor was verified from the installed macro, while another ROS distribution could change that generated implementation [UNCLEAR — verify].
- The Patchwork++ paper and external datasets are not embedded source material; algorithm semantics here are documented from the bundled implementation only.

## Glossary

| Term | Meaning in this package |
| --- | --- |
| Patchwork++ | Self-adaptive 3D point-cloud ground segmentation algorithm wrapped by this package. |
| GSeg | Abbreviation for ground segmentation; used in the `gseg_component` target name. |
| CZM | Concentric Zone Model, the radial zone/ring/sector partition used before local plane fitting. |
| PCA | Principal Component Analysis, used to estimate a local plane normal. |
| LPR | Lowest Point Representative, the low-point sample used to seed ground estimation. |
| GLE / A-GLE | Ground Likelihood Estimation / Adaptive GLE, checks based on uprightness, elevation, and flatness. |
| R-GPF | Region-wise Ground Plane Fitting. |
| R-VPF | Region-wise Vertical Plane Fitting, enabled in the core defaults. |
| RNR | Reflected Noise Removal; forcibly disabled by the ROS wrapper. |
| TGR | Temporal Ground Revert, the core's adaptive reconsideration of candidate ground patches. |
| `base_frame` | Output header frame label for classified clouds; it does not trigger a TF transform. |
| Component | A node packaged as a loadable shared-library plugin for an `rclcpp_components` container. |
