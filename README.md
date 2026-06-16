# Skymind

Foundation-model-guided UAV exploration. Replaces a classical geometric frontier planner with a vision-language model (VLM) to study whether semantic reasoning improves exploration efficiency in unknown indoor environments, evaluated against a frontier-exploration baseline in AirSim.

## Status

Design and planning stage — implementation has not started yet.

## Documentation

- [Design Spec](docs/specs/2026-06-15-skymind-design.md) — architecture, components, evaluation protocol, novelty positioning, and related work
- [Implementation Plan](docs/plans/2026-06-15-skymind.md) — phased task breakdown

## Approach

Three ROS 2 layers communicating over topics: a perception layer (scene description + occupancy mapping), a VLM planner that outputs the next waypoint, and an execution layer (safety filter + MAVROS bridge to PX4). A geometric frontier planner serves as the controlled baseline for A/B comparison.

See the design spec for full system architecture, related work, and the experimental protocol.
