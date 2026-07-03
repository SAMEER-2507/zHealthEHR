# Lidar-based Mapless Navigation — Writeup

## Approach

The baseline navigator was pure greedy-toward-goal with no planning, so it
got stuck behind any wall it hadn't seen yet (27% success). I replaced it
with **frontier-based exploration + BFS replanning**, run fresh every step
since the grid is small (40x40, BFS over ≤1600 cells is cheap):

1. **Update the known map** from the lidar scan (free/wall per cell).
2. **BFS from the robot's pose** over cells confirmed `FREE` only. This
   gives, for free, both (a) the set of reachable known-free cells and
   (b) a shortest path to any of them, including the goal if it's been seen.
   Because the path only ever crosses cells already confirmed free, it
   cannot collide with a wall.
3. **Frontiers** = reachable free cells with at least one unexplored
   neighbor. While the goal isn't confirmed reachable yet (or there's still
   unexplored area worth covering), the robot heads to the *nearest*
   frontier, tie-broken toward whichever unexplored frontier is also closer
   to the goal (`score = dist_from_robot + 0.4 * manhattan_dist_to_goal`).
   This biases exploration to happen roughly along the route to the goal
   instead of wandering in the opposite direction.
4. A frontier target is **sticky** (kept until reached or invalidated) so
   the robot doesn't flip-flop between two similarly-scored frontiers.
5. Once **no frontiers remain reachable** (map fully explored) — or the
   step budget is getting tight relative to the known shortest path to the
   goal — the robot switches to walking straight to the goal via the BFS
   path.
6. **Fallback**: if no known-free path exists at all yet (e.g. the very
   first couple of steps, before any scan), it falls back to a local
   greedy move that prefers stepping into unseen cells, same spirit as the
   original baseline.

I also fixed a subtle bug in the given scan-processing order: the starter
code let a `FREE` classification silently overwrite an already-recorded
`WALL`. I flipped it so wall hits always win on conflict — cheap, and
avoids planning a "safe" path straight through an actual wall.

## Balancing goal-reaching vs. coverage

Since scoring is `success > coverage > SPL`, and success is a hard gate
(0 score on failure), the first priority was guaranteeing success: only
ever move over cells with confirmed free status (steps 2–3 above), so
collisions can't strand the robot in a way that risks the deadline.

Given success is basically guaranteed, coverage was maximized by fully
draining frontiers before beelining to the goal — this consistently hits
94–100% coverage on the 40x40 test maps. That drags SPL down (the robot
takes a scenic route while mapping), but the score weights
(0.65 coverage vs 0.35 SPL) mean this trade is favorable. I swept the
goal-bias weight in the frontier score (0.1 → 2.0); values that push much
harder toward "beeline first, explore incidentally" raise SPL but drop
coverage by 15–25 points and net a *lower* combined score. 0.4 gave the
best balance on the 15 training seeds (success 100%, coverage 96%,
SPL 0.23, combined score ~0.70, vs. 0.18 for the baseline).

## Known limitations / what I'd change with more time

- The switch-to-direct trigger uses a hardcoded assumption of a 1000-step
  budget (matching `run.py`'s default). If graded with a much smaller
  budget, the full-exploration strategy could run out of steps before
  reaching the goal. A more robust version would detect exploration
  plateau (known-free region stops growing) instead of relying on an
  assumed budget number.
- BFS is recomputed from scratch every step rather than incrementally
  updated — fine at this grid size (a few hundred ms per full run) but
  wouldn't scale to much larger maps.
- The frontier goal-bias weight (0.4) was tuned by sweeping on the 15
  training seeds; it's a reasonable single value but not adaptive to map
  shape (e.g. long narrow corridors vs. open rooms might want different
  bias).
- No SLAM-style pose uncertainty is modeled — pose is assumed exact, per
  the environment's API.
