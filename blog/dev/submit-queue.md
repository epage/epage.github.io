---
title: Submit Queue
---

Time keeps marching on while developing a branch for merging and history
diverges.  `git` merge history but fails when similar lines are changed,
conflicting with each other.  Most Code Review tools will report there would be
one of these line conflicts.  What they can't do is report when there would be
semantic conflicts (e.g. adding new code that calls a function that was renamed).
Only running CI will detect these types of conflicts.

Strategies for reducing the impact of semantic conflicts include:
- CI runs against the developer's branch:
  - Merge race window: Between last manual rebase and merge-time.
  - Cost: none
  - User notification: post-merge
- CI runs against the developer's branch, merged with `master`:
  - Merge race window: Between last push and merge-time.
  - Cost: none
  - User notification: post-push
- Code Review policy rejects PRs that are X days old or Y commits behind:
  - Merge race window: bounded by configuration
  - Cost: compute time and/or accept latency due to "satisfy the tool" pushes
  - User notification: after X time
- Merges into `master` are serialized and only made available when CI passes:
  - Merge race window: none
  - Cost: compute time and/or merge latency from extra pipeline run
  - User notification: pre-merge

Other benefits to Submit Queues:

- To speed up PR feedback, its common to rely heavily on test avoidance but
  there are gaps where we don't have precise enough information and we err on
  the side of faster CI runs rather than more complete CI runs.  Sometimes, this is worked around by markers the user can leave to get extra validation.
  A Submit Queue gives us the best of all worlds: fast PR feedback and green-master.
- There is a gap between when a PR lands and CI pushing out artifacts.  Submit Queue artifacts could be staged and made available if the pipeline succeeds, making them available immediately on merge.

Considerations

- Emergency fixes
- Uncaught failures in the Submit Queue CI run, potentially failing unrelated changes in the future
- CI Flakiness rejecting changes unnecessarily and disrupting any speculative CI runs
- Latency before changes become available
- Controlling for side effects in Submit Queue CI runs since they might not land (e.g. deployments)
- Improve throughput through predicting failure
  - ML like Uber
  - PRs too old, like Shopify
- Resource utilization
  - Required processing "bandwidth"
  - Cost

Existing Submit Queues

- [bors](https://bors.tech/)
  - Used by Rust and other projects
  - Integrates with GitHub
  - [Communicate through @-mentions in PR comments](https://bors.tech/documentation/)
  - Can request a full test suite run without a merge
  - Uses a priority queue
  - [Batches changes](https://github.com/bors-ng/bors-ng#how-it-works)
    - Batch per priority level
    - Only one batch is active at a time
      - Max on success latency is O(2 * ci_time)
      - Max system load is O(pipeline)
      - Cost is O(pipeline * ci_time)
    - Bisects batches on failure
      - Latency is O(num_errors * log queue_size)
      - Max system load is O(pipeline)
      - Cost is O(pipeline * ci_time * num_errors * log queue_size)
- [Gitlab Merge Trains](https://docs.gitlab.com/ee/ci/merge_request_pipelines/pipelines_for_merged_results/merge_trains/)
  - Communicate through MR UX
  - Has "Land immediately" button, jumping the queue
  - Currently conflicts with "Merge When Pipeline Succeeds" feature
  - Speculative execution, rather than batching
    - On success latency is always O(ci_time)
    - Max system load is O(queue_size * pipeline)
    - Cost is O(queue_size * pipeline)
  - Restarts process on failure
    - Latency is O(num_errors * ci_time)
    - Max system load is O(queue_size * pipeline)
    - Cost is O(pipeline * ci_time * num_errors * queue_size)
- [ShipIt Merge Queue](https://github.com/Shopify/shipit-engine#merge-queue)
  - Used by Shopify
    - See also [v1 announcement](https://shopify.engineering/introducing-the-merge-queue) and [v2 announcement](https://shopify.engineering/successfully-merging-work-1000-developers)
  - Integrates with Github
  - Auto-rejects stale PRs (age and #commits)
  - Communicate through @-mentions in PR comments
  - Has "Land immediately" command, jumping the queue
  - Queue locking for emergencies
    - Auto-lock on too many items in queue
  - Batching
    - Fixed size queue (they use 8) to balance throughput and fault isolation
    - Batches run in parallel (speculative execution?).  They have it fixed at 3 to limit strain on CI resources.
- [arc submit](https://github.com/uber/arcanist/blob/master/src/workflow/ArcanistSubmitWorkflow.php)
  - Used by Uber (forked arc)
    - [Paper](https://eng.uber.com/research/keeping-master-green-at-scale/) ([web version?](https://blog.acolyer.org/2019/04/18/keeping-master-green-at-scale/))
  - Speculation engine tries to predict build success
    - Graph is created with priority, success, and if builds two merges can safely be done at once
