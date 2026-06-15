# Design Spec: Foundation-Model-Guided UAV Navigation
**Date:** 2026-06-15
**Author:** Arjav Singh
**Target:** ETH Zurich Master's Application (November 2026)

---

## 1. Problem Statement

Most foundation-model-guided navigation research targets ground robots operating in 2D space. This project adapts the VLM-as-Planner paradigm to aerial robots (UAVs), introducing 3D spatial reasoning, hovering dynamics, and altitude-aware scene understanding as novel challenges. The core question: **does replacing a classical geometric frontier planner with a vision-language model improve exploration efficiency in unknown indoor environments?**

---

## 2. Novelty Claim

> "We present the first systematic adaptation of VLM-based semantic exploration to UAV platforms, evaluating it against classical frontier exploration across coverage efficiency, collision rate, and semantic accuracy in AirSim simulation, with a real-world deployment attempt."

This is directly comparable to ground-robot baselines (NavGPT, VoroNav) while introducing aerial-specific challenges that prior work has not addressed.

---

## 3. System Architecture

Three loosely-coupled layers communicating over ROS 2 topics:

```
[AirSim + PX4 SITL]
    ↓ /camera/rgb, /camera/depth, /mavros/local_position/pose
[Layer 1: Perception]
    Scene Describer (LLaVA/BLIP-2)   →   natural language scene description
    Occupancy Mapper (depth + pose)   →   2D explored/unexplored grid
    ↓ scene_description (String), occupancy_map (OccupancyGrid)
[Layer 2: VLM Planner]
    LLaMA-3-Vision (local) or GPT-4V (API)
    Input:  scene description + occupancy map + UAV pose + task goal
    Output: next waypoint (x, y, z, yaw) + reasoning string
    ↓ /planner/waypoint (PoseStamped)
[Layer 3: Execution]
    Safety Filter   →   validates waypoint against occupancy map
    MAVROS bridge   →   PX4 OFFBOARD mode position setpoints
    ↑ completion feedback → triggers new perception cycle
```

### ROS 2 Nodes

| Node | Subscribes | Publishes |
|------|-----------|-----------|
| `scene_describer` | `/camera/rgb` | `/perception/scene_description` |
| `occupancy_mapper` | `/camera/depth`, `/mavros/local_position/pose` | `/perception/occupancy_map` |
| `vlm_planner` | `/perception/scene_description`, `/perception/occupancy_map`, `/mavros/local_position/pose` | `/planner/waypoint`, `/planner/reasoning` |
| `safety_filter` | `/planner/waypoint`, `/perception/occupancy_map` | `/execution/safe_waypoint` |
| `waypoint_controller` | `/execution/safe_waypoint` | `/mavros/setpoint_position/local` |

---

## 4. Components

### 4.1 Scene Describer
- **Model:** LLaVA-1.6 (7B, local) as primary; BLIP-2 as lightweight fallback
- **Input:** RGB frame at ~2 Hz
- **Output:** 1–3 sentence description focused on spatial layout and navigability
- **Prompt template:** `"Describe this UAV camera view for navigation. Focus on: open spaces, obstacles, doors, corridors, unexplored areas. Be concise."`
- **Implementation:** Python ROS 2 node wrapping `ollama` or `transformers` inference

### 4.2 Occupancy Mapper
- **Input:** Depth image + drone pose (from MAVROS)
- **Output:** 2D top-down occupancy grid (explored / unexplored / obstacle)
- **Implementation:** Project depth pointcloud to 2D grid; mark cells as explored when drone passes within 1.5m; obstacles from depth thresholding
- **Library:** `octomap_server` or custom numpy grid

### 4.3 VLM Planner
- **Model:** LLaMA-3.2-Vision-Instruct (11B) locally via `ollama`; GPT-4V via API as fallback
- **Prompt structure:** Few-shot chain-of-thought with 3 examples of good exploration decisions
- **Output parsing:** Extract `(x, y, z, yaw)` from structured JSON in model output; retry once on parse failure; fall back to hover on second failure
- **Planning frequency:** Once per waypoint completion (event-driven, not fixed-rate)

### 4.4 Safety Filter
- **Checks:** (1) waypoint within arena bounds, (2) path to waypoint collision-free in occupancy map, (3) altitude within [0.5m, 3.0m]
- **On rejection:** request new waypoint from planner with rejection reason in context
- **Max retries:** 3 before falling back to hover-in-place

### 4.5 Waypoint Controller
- **Interface:** MAVROS `/mavros/setpoint_position/local` in OFFBOARD mode
- **Speed limit:** 0.5 m/s in simulation, 0.3 m/s in real-world
- **Completion criterion:** position error < 0.2m AND yaw error < 5°
- **Reuse:** Directly extends existing PX4 controller from `skynet-ws`

---

## 5. Simulation Environment

- **Simulator:** AirSim (Unreal Engine) — already set up in SITL project
- **Scene:** Indoor warehouse or hospital map with corridors, rooms, doors
- **Flight controller:** PX4 SITL via Docker (existing setup)
- **ROS bridge:** MAVROS (existing in skynet-ws)
- **Arena size:** ~20m × 20m × 3m (indoor scale)

---

## 6. Baseline

Classical frontier exploration (geometric only):
- Same hardware stack and ROS 2 nodes
- Layer 2 (VLM Planner) replaced with standard frontier-based planner
- Frontier = centroid of nearest unexplored boundary cell in occupancy grid
- No semantic reasoning — purely geometric

This gives a clean A/B comparison: **semantic VLM planning vs geometric frontier planning**, same everything else.

---

## 7. Evaluation

### Metrics
| Metric | Definition |
|--------|-----------|
| Coverage % | Fraction of navigable area explored after N minutes |
| Time-to-X% | Time to reach 50%, 75%, 90% coverage |
| Collision rate | Collisions per trial (lower is better) |
| Planner failures | Times safety filter rejected all retries |
| Semantic accuracy | Human-rated: does the VLM description match the scene? (1–5 scale, 20 samples) |

### Experimental Protocol
- 20 trials per condition (VLM planner, frontier baseline, random walk)
- 3 different indoor maps to test generalization
- Time limit: 10 minutes per trial
- Metrics logged to rosbag + CSV; plotted with matplotlib/seaborn

### Ablations
- Local LLM (LLaMA-3.2) vs API (GPT-4V)
- With scene description vs without (waypoint only from map)
- Prompt variants: zero-shot vs few-shot chain-of-thought

---

## 8. Real-World Deployment (Phase 5, Optional)

- **Hardware:** Existing drone from skynet-ws project
- **Sensor swap:** Replace AirSim camera topics with real RealSense D435 topics (same ROS 2 interface)
- **Safety:** Reduced speed (0.3 m/s), manual override always armed, indoor controlled space
- **Goal:** At least one successful full-room exploration run
- **Fallback:** If hardware unavailable, document sim-to-real gap analysis from literature

---

## 9. Timeline

| Phase | Dates | Deliverable |
|-------|-------|-------------|
| 1. Setup | Jun 15 – Jul 6 | AirSim scene + VLM inference working |
| 2. Perception | Jul 7 – Jul 27 | Scene describer + occupancy mapper nodes |
| 3. Planning | Jul 28 – Aug 24 | Full closed loop + baseline |
| 4. Evaluation | Aug 25 – Sep 21 | Results, plots, failure analysis |
| 5. Real-world | Sep 22 – Oct 19 | Real flight attempt (optional) |
| 6. Write-up | Oct 20 – Nov 2 | GitHub + demo video + technical report |

---

## 10. Knowledge Requirements & Resources

### Phase 1
**What to know:** ROS 2 nodes, topics, launch files; running inference with HuggingFace/ollama
**Resources:**
- [ROS 2 Humble Tutorials](https://docs.ros.org/en/humble/Tutorials.html) — nodes, topics, colcon build
- [Ollama quickstart](https://ollama.com/library/llava) — run LLaVA locally in one command
- [AirSim ROS2 wrapper](https://github.com/microsoft/AirSim/blob/main/docs/ros2.md)

### Phase 2
**What to know:** ROS 2 image processing (`cv_bridge`, `sensor_msgs`); pointcloud basics; numpy grid operations
**Resources:**
- [cv_bridge tutorial](https://wiki.ros.org/cv_bridge/Tutorials/ConvertingBetweenROSImagesAndOpenCVImagesPython)
- [sensor_msgs/PointCloud2 with Open3D](https://github.com/isl-org/Open3D-ROS) — depth → pointcloud
- [OctoMap ROS](https://octomap.github.io/) — 3D occupancy mapping

### Phase 3
**What to know:** Prompt engineering for structured output; JSON parsing from LLM responses; MAVROS OFFBOARD mode
**Resources:**
- [Prompt engineering guide (Anthropic)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — chain-of-thought, structured output
- [MAVROS OFFBOARD mode tutorial](https://docs.px4.io/main/en/ros/mavros_offboard_python.html)
- [NavGPT paper](https://arxiv.org/abs/2305.16986) — ground-robot VLM planner to adapt from
- [VoroNav paper](https://arxiv.org/abs/2309.10329) — semantic frontier exploration baseline

### Phase 4
**What to know:** Experimental design; matplotlib/seaborn plotting; statistical significance (t-test)
**Resources:**
- [Matplotlib tutorials](https://matplotlib.org/stable/tutorials/index.html)
- [scipy.stats.ttest_ind](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ttest_ind.html) — compare means between conditions
- [How to write a robotics evaluation section](https://arxiv.org/abs/2304.01196) — CoNAV as example

### Phase 5
**What to know:** Sim-to-real transfer; RealSense D435 ROS 2 driver; safety considerations for real flight
**Resources:**
- [Intel RealSense ROS2 wrapper](https://github.com/IntelRealSense/realsense-ros)
- [PX4 safety configuration](https://docs.px4.io/main/en/config/safety.html)

### Phase 6
**What to know:** LaTeX (IROS/NeurIPS template); how to structure a technical report
**Resources:**
- [IROS 2026 paper template](https://www.iros25.org/call-for-papers) — standard robotics venue format
- [How to write a research paper (Whitesides)](https://intra.ece.ucr.edu/~rlake/Whitesides_writing_res_paper.pdf)

---

## 11. Repository Structure

```
skymind/
├── ros2_ws/
│   └── src/
│       ├── perception/          # scene_describer, occupancy_mapper nodes
│       ├── planning/            # vlm_planner, safety_filter nodes
│       ├── execution/           # waypoint_controller node
│       └── evaluation/          # metrics logging, trial runner
├── scripts/
│   ├── run_simulation.sh        # launch AirSim + PX4 + ROS 2 stack
│   ├── run_baseline.sh          # same but with frontier planner
│   └── run_eval.sh              # batch trial runner
├── config/
│   ├── prompts/                 # VLM prompt templates (YAML)
│   └── params/                  # node parameters (YAML)
├── evaluation/
│   ├── results/                 # trial CSVs
│   └── plots/                   # generated figures
├── docs/
│   └── superpowers/specs/       # this file
└── README.md
```

---

## 12. Resume / SOP Bullet

> **Foundation-Model-Guided UAV Exploration** *(Jun–Nov 2026)*
> Adapted VLM-as-Planner navigation (NavGPT/VoroNav paradigm) to aerial robots. Built a 3-layer ROS 2 system integrating LLaVA scene understanding with PX4 flight control in AirSim simulation. Showed X% improvement in coverage efficiency over classical frontier exploration across 20 trials on 3 indoor maps. Deployed to real hardware with RealSense D435 sensing.
