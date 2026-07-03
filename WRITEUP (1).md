# Lidar-based Mapless Navigation - Writeup

## Approach

The original navigator simply moved toward the goal without planning, which caused it to get stuck behind walls it hadn't encountered (27% success). I replaced it with **frontier-based exploration and BFS replanning**, refreshed every step given the small grid (40x40, BFS over ≤1600 cells is efficient):

1. **Update the known map** based on the lidar scan (free/wall per cell).
2. **BFS from the robot's position** over cells confirmed as `FREE`. This step provides both (a) the set of reachable known-free cells and (b) the shortest path to any of them, including the goal if it has been seen. Since the path only crosses cells confirmed as free, it cannot hit a wall.
3. **Frontiers** are reachable free cells that have at least one unexplored neighbor. While the goal isn't confirmed reachable or there is still unexplored area, the robot moves to the *nearest* frontier, favoring any unexplored frontier closer to the goal (`score = dist_from_robot + 0.4 * manhattan_dist_to_goal`). This method encourages exploration in the direction of the goal rather than wandering off.
4. A frontier target is **sticky** (held until reached or invalidated) so the robot does not switch back and forth between two similarly scored frontiers.
5. After **no frontiers remain reachable** (the map is fully explored) or if step limits are tight compared to the known shortest path to the goal, the robot goes straight to the goal using the BFS path.
6. **Fallback**: if no known-free path exists (e.g., in the first few steps before any scan), it defaults to a local greedy move that prefers entering unseen cells, similar to the original approach.

I also fixed a subtle bug in the scan-processing order. The starter code allowed a `FREE` classification to overwrite an already recorded `WALL` without notice. I changed it so that wall hits always take priority in conflicts. This approach is efficient and prevents planning a "safe" path that runs through an actual wall.

## Balancing goal-reaching vs. coverage

Since the scoring is `success > coverage > SPL`, with success being a strict requirement (0 score on failure), the priority was to ensure success. The robot only moves over cells with confirmed free status (steps 2–3), preventing collisions that could trap it and risk missing deadlines.

With success essentially guaranteed, coverage was maximized by fully exploring frontiers before heading to the goal. This consistently achieves 94–100% coverage on the 40x40 test maps. Although this may lower SPL (the robot takes a longer route while mapping), the score weights (0.65 for coverage vs. 0.35 for SPL) make this trade-off worthwhile. I adjusted the goal-bias weight in the frontier score from 0.1 to 2.0. Values that push more aggressively toward "go straight to the goal, explore later" might improve SPL but reduce coverage by 15–25 points and ultimately result in a *lower* combined score. A weight of 0.4 provided the best balance across 15 training runs (100% success, 96% coverage, SPL of 0.23, and a combined score of ~0.70, compared to 0.18 for the baseline).

## Known limitations / what I'd change with more time

- The trigger for switching to direct movement relies on a fixed assumption of a 1000-step budget (matching `run.py`'s default). If tested with a much smaller budget, the full-exploration strategy could exhaust its steps before reaching the goal. A better version would identify when exploration plateaus (when the known-free area stops expanding) rather than relying on a set budget.
- BFS is re-calculated from scratch every step instead of being updated incrementally. This is fine for the current grid size (a few hundred milliseconds per full run), but it wouldn't work well for larger maps.
- The frontier goal-bias weight (0.4) was tuned based on the 15 training runs. While it's a reasonable single value, it doesn't adjust to different map shapes (for instance, long narrow corridors versus open rooms might require different biases).
- No SLAM-style pose uncertainty is taken into account; the pose is assumed to be exact according to the environment's API.
