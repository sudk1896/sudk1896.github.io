---
title: Thread Questions from https://www.ops-class.org/asst/1/
layout: post
comments: false
---
1. What happens to a thread when it calls thread_exit? What about when it sleeps?

On thread_exit, thread structures are cleaned up and thread switches to ZOMBIE state.

2. What function or functions handle(s) a context switch?

thread_switch.

3. What does it mean for a thread to be in each of the possible thread states?

S_RUN = Running

S_READY = Ready to Run

S_SLEEP = Sleeping

S_ZOMBIE = exited but not yet deleted.
  
