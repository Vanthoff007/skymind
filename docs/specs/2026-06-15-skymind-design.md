# Skymind: Foundation-Model-Guided UAV Navigation
**Date:** 2026-06-15
**Author:** Arjav Singh

---

## Problem Statement

Right now, drones explore unknown indoor spaces (rooms, corridors) using a "geometric frontier planner" — a dumb algorithm that just flies toward the nearest unexplored patch on a map, with no understanding of what's actually in the scene. It doesn't know a corridor from a closet; it just sees "explored" vs "unexplored" cells.

The idea: replace that dumb planner with a vision-language model (VLM) — something like LLaVA or GPT-4V — that can actually *look* at the camera feed and *reason* about it ("this looks like a corridor, probably leads somewhere; that's a dead-end closet, skip it"). The hope is this semantic understanding makes exploration faster and smarter than blind geometry.

**Core question:** does giving a drone a VLM as its "brain" for deciding where to fly next actually explore an unknown indoor space better than the classic dumb geometric approach?

Most existing work on this idea (VLM-as-planner) has been done on ground robots. Adapting it to a drone adds new wrinkles: the drone moves in full 3D (not just along the floor), has to deal with altitude and hovering, and has tighter constraints on how fast it can react.

---

## Related Work (Reference Papers)

Closest prior art — read these first, since they're nearly the same idea one platform removed (ground robot, not aerial):

- **Autonomous Frontier-Based Exploration with VLM Guidance** — [arXiv:2605.23165](https://arxiv.org/abs/2605.23165). A VLM picks which frontier (unexplored boundary) to fly toward next, using the occupancy map + photos of each candidate direction, instead of a geometric heuristic. Ground robot, +24% map coverage over baselines, tried a real-world test too.
- **Efficient Navigation in Unknown Indoor Environments with VLMs** — [arXiv:2510.04991](https://arxiv.org/html/2510.04991v2). VLM looks directly at the occupancy grid image (not the camera feed) to rank candidate subgoals. ~10% shorter paths than baseline.

Aerial + VLM (different specifics, but same general space):

- **AirHunt** — [arXiv:2601.12742](https://arxiv.org/html/2601.12742). Drone + VLM for outdoor object search. Useful for how they solved the "VLM is slow, drone flight control needs to be fast" timing mismatch.
- **SoraNav** — [arXiv:2510.25191](https://arxiv.org/abs/2510.25191). Drone + VLM with a geometric fallback when the VLM gives a bad/unreachable answer — relevant for building a safety check around the VLM's output.

Foundational VLM-as-planner papers (ground robot, general paradigm):

- **NavGPT** — [arXiv:2305.16986](https://arxiv.org/abs/2305.16986)
- **VoroNav** — [arXiv:2309.10329](https://arxiv.org/abs/2309.10329)

---

## What to Learn

**Robotics / ROS 2**
- [ROS 2 Humble Tutorials](https://docs.ros.org/en/humble/Tutorials.html) — nodes, topics, how packages talk to each other
- [cv_bridge tutorial](https://wiki.ros.org/cv_bridge/Tutorials/ConvertingBetweenROSImagesAndOpenCVImagesPython) — converting camera images between ROS and OpenCV
- [MAVROS OFFBOARD mode tutorial](https://docs.px4.io/main/en/ros/mavros_offboard_python.html) — how to actually send the drone a "fly here" command

**Simulation**
- [AirSim ROS2 wrapper](https://github.com/microsoft/AirSim/blob/main/docs/ros2.md) — connecting the simulator to ROS 2
- [PX4 safety configuration](https://docs.px4.io/main/en/config/safety.html) — flight controller safety limits

**Mapping**
- [sensor_msgs/PointCloud2 with Open3D](https://github.com/isl-org/Open3D-ROS) — turning depth images into 3D points
- [OctoMap ROS](https://octomap.github.io/) — standard library for building an occupancy map (explored/unexplored/obstacle grid)

**VLM / LLM side**
- [Ollama quickstart](https://ollama.com/library/llava) — running a vision model like LLaVA locally, one command
- [Prompt engineering guide (Anthropic)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — getting structured, reliable output out of a model (you'll need this to get clean waypoints back instead of rambling text)

**Evaluation**
- [Matplotlib tutorials](https://matplotlib.org/stable/tutorials/index.html)
- [scipy.stats.ttest_ind](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ttest_ind.html) — checking if your VLM result is actually better than the baseline, statistically, not just by luck
