# HeadsetWithCam — Agent Working Rules

Read [README.md](README.md) and the `docs/` files before changing implementation.

## Current truth (2026-10-14)

Research-only repository; no verified E900 device, BLE recording API, Wi-Fi transfer protocol, or deployed iPhone bridge is claimed. Source documents:

- [Product / shopping research](docs/E900_RESEARCH_2026-10-14.md)
- [R08 + E09 protocol leads](docs/PROTOCOL_RESEARCH.md)
- [Architecture and validation gates](docs/ARCHITECTURE_AND_TEST_PLAN.md)
- [Agent handoff, task prompt, Definition of Done](docs/AGENT_HANDOFF.md)

## Instructions

1. Build `MockCameraDriver` and tests before any E900 live write commands.
2. Never assume a command that works on E09 also works on E900; label all such steps `UNVERIFIED`.
3. Separate `REFERENCE`, `MOCK PASS`, `HARDWARE PASS`, `BLOCKED`, and `NOT TESTED` in every report.
4. Do not run OTA, factory reset, media deletion, formatting, brute-force writes, or secret extraction.
5. State whether tests ran and show raw reproducible evidence; don't invent build logs, devices or file captures.
6. Respect Halo-Ring upstream AGPLv3 / dual-license notices if adapting copyrighted code; don't silently vendor it.
7. Avoid tracking personal recordings, raw serial numbers, physical BLE MAC addresses, access keys or credentials in Git.
8. Do not claim iOS background BLE is guaranteed to run after a user force-quits the app.
9. Keep user-visible recording indicators and privacy/consent considerations explicit.
10. Summarize any modifications and update docs/evidence and test/status matrix with verified outcomes.

Primary functional goal: R08 gesture -> iPhone CoreBluetooth bridge -> E900 video start/stop -> Wi-Fi media transfer -> Mac mini AI pipeline.
