# CarND-Path-Planning-Project

_Technologies: Path Planning, C++_

Built a path planner that navigates a vehicle through traffic on a highway. Used the concepts taught in the module - environmental prediction, behavioral planning, and trajectory generation - to build the planner.

_Part of the Self-Driving Car Engineer Nanodegree Program_

## General approach

My path planning algorithm follows these steps:

1. Use sensor fusion data to predict where the ego car and other cars will be in the future ("prediction module")
* First of all, the future position of the car is set as the last point from the previously generated path (end_path_s) - see lines 261-264
* Based on sensor fusion data, the algorithm checks if - in the "future" - there will be any car ahead within the range of 30m (line 282)
* If yes, then the ego car slows down to the speed slightly below the speed of the car ahead to avoid collision (lines 524 - 530)
* At the same time, the behavior planning "module" is started to evaluate potential lane change (see below)

2. Decide which lane (= state) is the best to be in ("FSM-based behavior planner")
* The behavior planner logic is implemented in lines 289 - 421
* In my behavior planner I define FSM states simply as target lanes, i.e. my behavior planner chooses the best lane to be in
* The behavior planner calculates cost for each lane
* There are three cost functions that contribute to overall cost of each lane:
- speed_penalty: the lower the speed of vehicles in a particular lane (vehicles within range of 50 meters are checked), the higher the speed penalty (see line 313)
- lane_change_opportunities: this is a hard-coded cost, which assigns certain cost to lanes 0 and 2 - due to the fact that, if all else is equal, it's better to be in the middle lane (lane 1) than in the "side" lanes. The reason for this is that there are more lane change opportunities if the car is in the middle lane than if it's in one of the "side" lanes.
- collision_penalty: if there is a car within the range of {+15;-7} meters from the ego car, the lane gets a very high collision penalty cost, effectively ensuring that the ego car will not attempt to change into that lane (see lines 335 - 364)
* The ego car will then attempt to change into the lane with the lowest cost. However, if the lane with the lowest cost is "2 lanes away" (e.g. the vehicle is currently in lane 0 and lane 2 has the lowest cost), then the car will first attempt to change to the middle lane (checking for a collision risk first), then drive in that lane for 25 cycles (=0.5s) and only afterwards will change to the target lane (see lines 396 - 416 for lane selection logic). This approach is neccessary to avoid veering across two lanes, which could cause a collision with a vehicle in the middle lane and/or lead to exceeding acceleration and jerk limits.

3. Generate a trajectory to reach the desired lane (state) - see lines 430 - 556
* The trajectory is generated by first taking last 2 points from the previous path, adding to them 3 sparse points from the new path, then fitting a spline that goes through all 5 points, and finally using the spline to generate 50 dense path points which are then fed into the simulator.
* Trajectory smoothness is ensured by using spline, which fits a smooth chain of polynomials to the sparse points. Additionally, to avoid strange, agressive trajectories, the car is required to stay in a given lane for at least 25 cycles (as described under behavior planning above).
* Trajectory generation makes also a heavy use of transformations between map coordinates and car coordinates (see lines 484 - 491 and 546 - 548).

## Rubric requirements walk-through
Below I describe how I have addressed each of the rubric requirements needed for the project to successfully pass a Udacity review.
   
### The car drives according to the speed limit.
Requirement: The car doesn't drive faster than the speed limit. Also the car isn't driving much slower than speed limit unless obstructed by traffic.

* This is achieved by setting the reference speed at 49.5 mph, which is then used in a speed_penalty cost function as well as in acceleration / deccelaration logic (see lines 313 and 524 - 535, respectively).

### Max Acceleration and Jerk are not Exceeded.
Requirement: The car does not exceed a total acceleration of 10 m/s^2 and a jerk of 10 m/s^3.

* This is achieved by setting speed decrease / increase between two path points at a maximum of +/- 0.112 mph, which corresponds roughly to 0.05 m/s, which in turn - assuming that car passes two points in 0.02s - corresponds to max acceleration/deccelaration of 0.05 m/s / 0.02s = 2.5m/s2 (see lines 528 and 534). This is a rather conservative approach, but it works well for the project.

### Car does not have collisions.
Requirement: The car must not come into contact with any of the other cars on the road.

* This is achieved by collision penalty cost, which is assigned if there is a vehicle in the lane between 15 meters ahead and 7 meters behind the car (see lines 335 - 364).

### The car stays in its lane, except for the time between changing lanes.
Requirement: The car doesn't spend more than a 3 second length out side the lane lanes during changing lanes, and every other time the car stays inside one of the 3 lanes on the right hand side of the road.

* This is achieved by ensuring that the sparse points for the path are generated in the middle of the lane (see lines 472 - 474, the "2 + 4 * lane" formula). Afterwards, the spline generates a smooth path between the middle of one lane and the middle of the other lane (see also discussion about trajectory generation above).

### The car is able to change lanes
Requirement: The car is able to smoothly change lanes when it makes sense to do so, such as when behind a slower moving car and an adjacent lane is clear of other traffic.

* This requirement is covered by the behavior planner, please refer to the behavior planning description above.
