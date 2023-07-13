# commaSteeringControl

`commaSteeringControl` is a dataset of car steering information from ~12500 hours of driving with openpilot engaged. We control steering on most 
cars in openpilot using `steeringTorque`. This results in some lateral acceleration depending on both the car's internal vehicle dynamics and external factors (car speed, road roll, 
etc). Learning this relationship is essential to have a robust feedback loop and to follow the planned trajectory. `commaSteeringControl` is the largest controls dataset of its kind, spanning 
100+ platforms across 10+ brands.
![image](https://github.com/commaai/comma-steering-control/assets/1649262/57458e52-aea1-4b16-9f02-fa5a99559999)

## Timeline
- In [0.8.15](https://blog.comma.ai/0815release/#torque-controller),
we introduced a [new controller](https://github.com/commaai/openpilot/blob/master/selfdrive/controls/lib/latcontrol_torque.py) that leveraged the relationship between 
steering torque and lateral acceleration.
- In [0.9.0](https://blog.comma.ai/090release/#torqued-an-auto-tuner-for-lateral-control), we introduced 
[torqued](https://github.com/commaai/openpilot/blob/master/selfdrive/locationd/torqued.py), which learns the relationship online.
- In [0.9.2](https://blog.comma.ai/092release/#chevrolet-bolt-euv), we introduced a non-linear feed-forward function.
- There has been [extensive community effort](https://github.com/twilsonco/openpilot/tree/log-info) to improve the controller (speed-based relationships, using neural networks, etc)
- We are working on similar techniques to further improve steering controls  

We recognize that access to large, high-quuality datasets is a pre-requisite to improving controls. So, we hope this dataset aids and accelerates community effort to 
bring the best openpilot experience to all cars.

## Dataset
- Download the dataset from [HuggingFace](https://huggingface.co/datasets/commaai/commaSteeringControl/tree/main/data)
- Checkout the example notebook at `visualize.ipynb`
```
# Data Structure
data/
├── Platform 1
|   ├── Segment 1
|   ├── ...
|   └── Segment N
└── Platform M
    ├── Segment 1
    └── ...

|    | Fields                | Description                                                                      | Value Range     |
|---:|:----------------------|:---------------------------------------------------------------------------------|:----------------|
|  0 | t                     | Time                                                                             | [0, 60]         |
|  1 | latActive             | Is openpilot engaged?                                                            | {True, False}   |
|  2 | steeringPressed       | Is steering wheel pressed by the user?                                           | {True, False}   |
|  3 | vEgo                  | Forward velocity of the car (m/s)                                                | [0, ∞]          |
|  4 | aEgo                  | Forward acceleration of the car (m/s^2)                                          | [-∞, ∞]         |
|  5 | steeringAngleDeg      | Steering Angle (Deg)                                                             | [-∞, ∞]         |
|  6 | steer                 | Normalized steer torque                                                          | [-1, 1]         |
|  7 | steerFiltered         | Normalized, rate limited steer torque                                            | [-1, 1]         |
|  8 | roll                  | Road roll (rad)                                                                  | [-0.174, 0.174] |
|  9 | latAccelSteeringAngle | Lateral acceleration requested from the planner                                  | [-∞, ∞]         |
| 10 | latAccelDesired       | Lateral acceleration computed from the steering wheel angle and vehicle dynamics | [-∞, ∞]         |
| 11 | latAccelLocalizer     | Lateral acceleration from the localizer                                          | [-∞, ∞]         |
| 12 | epsFwVersion          | EPS firmware version                                                             | str             |

```
![image](https://github.com/commaai/comma-steering-control/assets/1649262/580ac4df-a84e-48ca-b1f6-8729d64905fb)



## Notes
- All values from different messages are interpolated and synced to time `t`
- In `torqued`, we assume that the gravity adjusted lateral acceleration has a linear dependence wrt. the steer command. We fit a Total-Least-Squares solution to obtain the factor. We also assume a error-dependant friction value (causes the hysteresis).
- Steering torque is normalized in openpilot (to get `steer`), and further rate limits are applied (to get `steerFiltered`). `steerFiltered` is the best input signal.
- The `latAccelSteeringAngle` is computed using the vehicle model from openpilot (it uses `roll`). This is the best signal to predict as `latAccelLocalizer` can be quite noisy.
- In reality (especially for some cars), the relationship is non-linear depending on vehicle speed, and is temporally correlated (there is a lag between the signals).
- There may be a lag in openpilot fully regaining steering control after `steeringPressed` which may have to be accounted for.
- In some platforms, cars with different `epsFwVersion` have dramatically different steering behaviour, although this is not common.
- Any algorithm that could be upstreamed to openpilot needs to be simple, fast, and reliable - similar to `torqued`, simple non-linear functions, or simple MLPs etc.

![image](https://github.com/commaai/comma-steering-control/assets/1649262/51f9a4cc-deb8-4ec4-b835-60ad014a7569)
