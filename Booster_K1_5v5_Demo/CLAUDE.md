# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

RoboCup humanoid soccer stack (5v5 kid-size). A ROS 2 (colcon) workspace that runs on Booster robots (Jetson). Three runtime nodes cooperate:

- `brain` — strategy. A BehaviorTree.CPP 4 tree ticks at 100 Hz on a dedicated thread, consuming vision + game-controller topics and driving the robot via `RobotClient`. Core class is `Brain` (`src/brain/include/brain.h`), composed of `BrainConfig` (static params), `BrainData` (runtime state), `BrainLog` (rerun telemetry), `BrainTree` (BT wrapper with shared blackboard), `BrainCommunication` (teammate + referee I/O), `Locator`, and `RobotClient`. BT nodes receive a raw `Brain*` and read/write `brain->config` / `brain->data` directly — blackboard entries exist but much state is shared via the `Brain` instance.
- `vision` — TensorRT-based detection + segmentation (`.engine` models under `src/vision/model/`). Publishes detections, line segments, depth — topics configured under `brain_node.ros__parameters.vision` (default subscribes to d-robotics StereoNet; RealSense config is commented out in `config.yaml`).
- `game_controller` — receives RoboCup GameController UDP and republishes as `/robocup/game_controller`.

Interface packages: `booster_msgs`, `booster_ros2_interface`, `robocup_ros2_interface`, plus per-node `msg/` directories. External deps expected installed system-wide: `booster_interface`, `vision_interface`, `game_controller_interface`, `behaviortree_cpp`, `rerun_sdk`, `backward_ros`, `yaml-cpp`, `Eigen3`, `OpenCV`, `tf2*`.

## Build

From workspace root:

```bash
./scripts/build.sh                 # full colcon build (symlink-install, parallel)
./scripts/build_brain.sh           # --packages-select brain (fast iteration)
./scripts/build_debug.sh           # Debug build with -g -fno-omit-frame-pointer
```

All three wrap `colcon build --symlink-install --parallel-workers $(nproc)` and pass `"$@"` through, so extra `--packages-select …` / `--cmake-args …` flags work. After a build, `source ./install/setup.bash` before running `ros2 launch …` manually.

Brain is C++17 (`src/brain/CMakeLists.txt`), and generates the in-package `brain/msg/Kick.msg` interface during its own build — changes to that msg require a brain rebuild.

## Run

Composite launchers under `scripts/` stop existing nodes (`./scripts/stop.sh` → `killall -9` of each node) then `nohup ros2 launch …` each package, writing per-node logs to `<workspace>/<name>.log`:

- `./scripts/start.sh` — production: vision + brain (game tree) + game_controller. Masks apt timers, kills `python3`/`update_manager`, runs `jetson_clocks`. Requires sudo.
- `./scripts/chase.sh` — brain with `tree:=chase`, comms + file log disabled.
- `./scripts/calibrate.sh` — brain with `tree:=calibrate`, comms disabled.
- `./scripts/assist.sh` — brain with `tree:=assist`.
- `./scripts/stop.sh` — kills `vision_node`, `brain_node`, `game_controller`, `sound_play_node`.
- `./scripts/start_brain.sh` / `start_vision.sh` / `start_game_controller.sh` — single-node foreground, pass-through args.

Running brain manually: `ros2 launch brain launch.py tree:=game pos:=left role:=striker sim:=false disable_log:=false disable_com:=false`. Behavior tree files live in `src/brain/behavior_trees/` (`game.xml`, `chase.xml`, `demo.xml`, + `subtrees/`); `tree:=foo` resolves to `behavior_trees/foo.xml`.

No automated test suite. Brain build enables `BUILD_TESTING` hooks but `ament_cmake_copyright_FOUND` / `ament_cmake_cpplint_FOUND` are forced TRUE, so linters are skipped.

## Configuration layering

Brain parameters come from three YAML sources merged in order (later overrides earlier), per `src/brain/launch/launch.py`:

1. `src/brain/config/config.yaml` — committed defaults (team_id, player_id, role, strategy thresholds, vision topics, rerun log targets, etc.).
2. `src/brain/config/config_local.yaml` — optional, gitignored per-robot override. Create this for machine-specific tweaks rather than editing `config.yaml`.
3. Launch-time overrides from `launch.py` args (`tree`, `pos`, `role`, `sim`, `disable_log`, `disable_com`) — highest precedence.

Vision has the same pattern: `vision.yaml` + optional `vision_local.yaml`, located via the `vision_config_path` launch arg (brain forwards its own `vision_config_path` to vision on production paths, defaulting to `/home/booster/Workspace/robocup_5v5demo/src/vision/config` in `start.sh`). `src/vision/config/vision.yaml.bak` is a backup, not active.

Camera/topic selection lives under `brain_node.ros__parameters.vision` in `config.yaml` — switching between RealSense and d-robotics StereoNet means swapping which block is commented, not code changes.

## Distribution

`distribution/build_package.sh` does a clean `colcon build`, stages `install/`, `scripts/`, `utils/`, `configs/`, plus `src/vision/{model,config}`, and bundles via `makeself` into `distribution/packages/robocup_<git_sha>.run`. `install.sh` wipes `/home/booster/Workspace/robocup` and reinstalls. Vision `.engine` files are shipped with the package — they are target-specific (Orin) TensorRT builds, not portable.

## Conventions

- Comments, log strings, and docs are predominantly Chinese. Preserve that language when editing adjacent code.
- `Brain` is passed as a raw pointer into BT nodes; do not copy `shared_ptr<Brain>` from within a node. New BT action/condition nodes follow the `Analyze`-style pattern in `brain_tree.h` (constructor takes `(name, config, Brain*)`, exposes `providedPorts()`).
- Runtime data belongs in `BrainData`, static config in `BrainConfig`. Do not add new top-level members on `Brain` itself.
- New brain `.cpp` files under `src/brain/src/` are picked up automatically via `file(GLOB SOURCE_FILES src/*.cpp)` — no CMakeLists edit needed, but a clean rebuild may be required for glob to re-scan.
