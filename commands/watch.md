---
description: Wait for CircleIt design feedback and handle each item as it arrives
---

Enter CircleIt watch mode: repeatedly call circleit_wait_for_feedback and handle each item using the circleit skill, until the user interrupts.

When a call times out with no feedback, call it again. If it reports that live delivery is on, tell the user that feedback already arrives in this session on its own, and stop.
