# commaSteeringControl

`commaSteeringControl` is a dataset of car steering measurements from ~12500 hours of driving with openpilot engaged. We control steering on most  cars in openpilot using `steeringTorque`. This results in some lateral acceleration depending on both the car's internal vehicle dynamics and external factors (car speed, road roll, etc). Learning this relationship is essential to having accurate steering control in openpilot. `commaSteeringControl` is the largest controls dataset of its kind, spanning 
hundreds of car models across 10+ brands.

The main purpose of this dataset is to give the community access to the data needed to model the steering of their car, and with that make a more accurate steering controller in openpilot to improve openpilot's performance on that car.

This is the largest dataset of vehicle dynamics ever released. It can also be used to develop or verify practical vehicle dynamics models for lateral acceleration, tire slip, road roll, understeer/oversteer, etc. We may add more fields for this goal in the future.

![image](https://github.com/commaai/comma-steering-control/assets/1649262/c6f18767-26ac-4bc8-ab60-afdae197a300)

## Dataset
- Download the dataset from [HuggingFace](https://huggingface.co/datasets/commaai/commaSteeringControl/tree/main/data)
- Checkout the example notebook at [`visualize.ipynb`](https://github.com/commaai/comma-steering-control/blob/master/visualize.ipynb)
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

![image](https://github.com/commaai/comma-steering-control/assets/1649262/f0195877-48ad-4664-85d6-7b2df12eb3d0)

## Dataset Notes
- All values from different messages are interpolated and synced to time `t`
- Steering torque is normalized in openpilot (to get `steer`), and further rate limits are applied (to get `steerFiltered`). `steerFiltered` is the best input signal.
- The `latAccelSteeringAngle` is computed from steering angle and roll using the vehicle model from openpilot. This is the best signal to predict as `latAccelLocalizer`, which comes from a sensor fusion localizer on the comma three device, can be quite noisy.
- In reality (especially for some cars), the relationship is non-linear depending on vehicle speed, and has temporal dynamics. On many cars the steering command is processed and smoothed inside the EPS causing non-linearities and temporal effects. There are also temporal effects in the physics (like in a mass-spring-damper model).
- There may be a lag in openpilot fully regaining steering control after `steeringPressed` which may have to be accounted for.
- In some platforms, cars with different `epsFwVersion` have dramatically different steering behaviour, although this is not common.
- Any algorithm that could be upstreamed to openpilot needs to be simple, fast, and reliable - similar to `torqued`, simple non-linear functions, or simple MLPs etc.

![image](https://github.com/commaai/comma-steering-control/assets/1649262/03905b06-6894-4b67-bd5b-77b1de552e62)

## Timeline of lateral control modeling in openpilot
- In [0.8.15](https://blog.comma.ai/0815release/#torque-controller),
we introduced a [new controller](https://github.com/commaai/openpilot/blob/master/selfdrive/controls/lib/latcontrol_torque.py) that leveraged the relationship between steering torque and lateral acceleration.
- In [0.9.0](https://blog.comma.ai/090release/#torqued-an-auto-tuner-for-lateral-control), we introduced 
[torqued](https://github.com/commaai/openpilot/blob/master/selfdrive/locationd/torqued.py), which learns the relationship online. Here we assume that the gravity adjusted lateral acceleration has a linear dependence wrt. the steer command. We fit a Total-Least-Squares solution to obtain the factor. We also assume an error-dependant friction value (causes the hysteresis).
- In [0.9.2](https://blog.comma.ai/092release/#chevrolet-bolt-euv), we introduced a non-linear feed-forward function.
- There has been [extensive community effort](https://github.com/twilsonco/openpilot/tree/log-info) to improve the controller (speed-based relationships, using neural networks, etc).
- We are working on further improvements for future releases. 
