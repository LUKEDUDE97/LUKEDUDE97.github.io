---
layout: post
title: 'Whisker-based Active Tactile Perception for Contour Reconstruction'
date: 2024-10-07 21:02 +0200

categories: [Eins-Project, Engineering Log]
tags: [Whisker, Tactile Servoing]

math: true
mermaid: true

post_description: A final version of manucript on whisker-inpired tactile sensing, which includes basic system intergation, tip contact estimation and active tactile servoing.
---

## I. Abstract

---

Perception using whisker-inspired tactile sensors currently faces a major challenge: the lack of active control in robots based on direct contact information from the whisker. To accurately reconstruct object contours, it is crucial for the whisker sensor to continuously follow and maintain an appropriate relative touch pose on the surface. This is especially important for localization based on tip contact, which has low tolerance for sharp surfaces and must avoid slipping into tangential contact. In this paper, we first construct a magnetically transduced whisker sensor, featuring a compact and robust suspension system composed of three flexible spiral arms. We develop a method that leverages a characterized whisker deflection profile to directly extract the tip contact position using gradient descent, with a Bayesian filter applied to reduce fluctuations. We then propose an active motion control policy to maintain the optimal relative pose of the whisker sensor against the object surface. A B-Spline curve is employed to predict the local surface curvature and determine the sensor orientation. Results demonstrate that our algorithm can effectively track objects and reconstruct contours with sub-millimeter accuracy. Finally, we validate the method in both simulations and real-world experiments where a robot arm drives the whisker sensor to follow the surfaces of three different objects.

## II. Introduction

---

Many animals use their vibrissae, or whiskers, to navigate and perceive their environment in dark, confined spaces. This tactile sensing complements optical perception, with the flexibility of whiskers allowing them to maneuver through various scenarios without obstruction. However, in most situations, whisker sensing is employed passively to help animals interact with complex environments. For example, mice leap over obstacles after sensing bumps around the lower areas of their rostral whiskers. Notably, some rats engage in sophisticated active sensing by rotating their vibrissae arrays to extract texture features and shapes from their surroundings. This active sensing behavior serves as the primary inspiration for the method we present here.

Whisker-inspired tactile sensing is also advantageous for enhancing robotic perception. For instance, navigating cluttered and unstructured environments requires robots to be acutely aware of nearby objects in close proximity, where their motion is often restricted and visibility severely limited due to occlusions between objects from optical sensors. In such scenarios, perception through multiple modalities, including touch, becomes crucial, and the whisker sensor emerges as an ideal solution, considering its flexibility and lightweight. The reaction force of contact from an object is transmitted into deflection along the whisker shaft and deformation on the root device, or embedded mechanoreceptors, which minimizes disturbance to any free-standing light objects and allows the robot to maneuver freely in tight spaces. Although this whisking system can accurately reconstruct surfaces by locating contact positions along the shaft, a significant challenge remains: Without active motion control to continuously follow the unknown surface, this reconstruction can only capture partial feature of the object which remains as a passive perception. While several notable studies have focused on applying active perception or tactile servoing to various other sensors, such as TacTip and the iCub fingertip, few have attempted to implement active control based on stimuli from whisker sensors. This gap serves as the primary focus of this paper.


![Fig. 1](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig1-1.png){: width="700" .shadow}
_Fig. 1. Demonstration of active tactile perception based on whisker-inspired sensor for contour reconstruction. The sensor is driven by a robot arm to continuously follow and contact with the unknown object surface._

Some previous research relies on tip contact of the whisker shaft to locate objects. While this approach cause even less disturbance on objects with light contact force, it is prone to detachment from the object or slipping into tangential contact, leading to a limited measurement range and low tolerance for sharp surfaces. Although tangential contact experiences less significant fluctuations, its trajectory is still strongly influenced by factors such as friction, texture, surface defects, and local curvature. Therefore, it becomes even more critical for the whisker sensor to continuously maintain an optimal contact pose against the object to ensure steady and accurate contact localization.

In this paper, we first develop a root suspension system with three integrated spiral arms to create a magnetically transduced whisker sensor that is more compact and robust. To test our active control strategy, we propose a direct method for extracting the 2D coordinates of the tip contact position, based on the characterized deflection profile of the whisker. This profile includes the known total length of the whisker shaft, a root position with rotation and linear translation at the center of the suspension device, and a calibrated whisker model built by sampling deflection measurements at various contact positions. The tip position is calculated using gradient descent, with fluctuations further reduced by applying a Bayesian filter, assuming a constant contact state. We then develop a combination of robot control and whisker-based tactile perception within a single loop To maintain the sensor's perceived orientation relative to the edge, a B-Spline curve is used to predict the local surface curvature, with a fixed total linear displacement between iterations. The sensor is driven to move tangentially, while adaptive adjustments are made to its normal displacement based on the deflection magnitude and a PID controller.

In short, the **key contributions** are as follows: 
- We construct a magnetically transduced whisker sensor based on a root suspension system with three integrated spiral arms, resulting in a more compact design (with a radius of only `3.36 mm`) and improved robustness. 
- We propose a direct method based on gradient descent to extract the 2D tip contact position with reduced fluctuations, achieving sub-millimeter accuracy (with an average distance of `0.08 mm`) and ease of computation.
- We develop a combination of active motion control and tactile perception within a single loop, using a B-Spline curve to enable the sensor to continuously follow the object's edge and reconstruct its entire surface. 

## III. Background and Related Work

---

### Whisker-inspired Tactile Sensor

Various studies have explored different structural designs for whisker sensors, with the key differences lying in their basic transduction principles—specifically, how the contact force from the whisker is transmitted into stimulus signals at the base. One of the most important designs is the magnetically transduced whisker, which uses a Hall effect sensor placed beneath an axially magnetized magnet and suspension system. This design is accurate, compact, and offers high angular deflection resolution, although this structure may suffer from fabrication tolerances. Another popular method is the MEMS barometer based whisker, where the whisker shaft is directly attached to the receptor surface and connected to barometers and a PCB layout. This approach is robust and easily integrated with other whiskers to form an sensory array. Additionally, 6-axis force/torque sensors are also widely used as transducers, providing accurate estimations of tangential contact positions based on the acquired bending moment. However, they are often too bulky and expensive for practical use in robotics. Optical-based approaches face similar challenges, requiring substantial mounting space and offering limited sensitivity.

Whisker sensors have proven useful to reconstruct the object contours~. It relies on an accurate and robust contact localization which however is extremely challenging for two main reasons: 1) Different contact states, varying reaction forces, and texture features can cause friction or slip, leading to unpredictable fluctuations in estimation; 2) Differentiating the actual position of a tangential contact, as shown in Fig.2B, is difficult because there is no direct, unique mapping from the single deflection measurement at the whisker root to a specific point along the shaft. Generally, tip contact causes more fluctuation than tangential contact due to friction, yet it is more common unless the whisker is overly deflected or contacts an object mid-shaft. Previous studies have also proposed solutions for tangential contact estimation, but these often require torque measurement with complex models to fit unique mapping or extra proprioceptive sensing.

### Active Tactile Perception

Many previous studies have focused on the development of various tactile sensors, enabling robots to navigate, perform complex manipulations, interact with environments and notably, actively reconstruct these environments in the absence of vision. For instance, Lepora et al. proposed a biomimetic active touch approach that integrates perception and tactile servoing into a single control loop, demonstrating the ability to track edges using tactile stimuli from the iCub fingertip and TacTip. Although numerous other exploration strategies have emerged recently, a significant challenge remains: direct contact could cause disturbance on object's original state, often leading to the use of intermittent tap contacts for sensing. By interpreting it as a SLAM problem, Suresh et al. was able to infer object shapes while simultaneously maintaining continuous contact and accounting for motion induced by tactile interaction.

Base on its flexibility and lightweight, Whisker sensor present as an another potential solution to the problem by providing continuous contact perception without disturbing objects; however, few studies have explored this avenue. Xiao et al. applied a MEMS-based whisker array to actively extract object features, but did so using a spiral movement with discrete contacts, and the whiskers were only used to detect collisions without actual contact localization. A recent study by Kossas et al. proposed a whisker-based navigation algorithm, but it primarily focused on integrating tactile sensation into the robot's motion decision-making process, without constructing actual maps.

## IV. Methodology

---

### Sensing System Integration

![Fig. 2](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig2-1.png){: width="700"  .shadow}
_Fig. 2. (A) Basic structure of our whisker-inspired tactile senor based on magnetically transduced principle. (B) The contact force on $p_c$ leads to a deflection along the whisker shaft, deformation on suspension device and rotation on magnet, ends up with a magnetic flux change measurement on Hall effect sensor. The tangential contacts at $p_a$, $p_b$ and $p_c$ result in an exact same deflection measurement on root._

The overall mechanism schematic of our magnetically transduced whisker sensor is illustrated in Fig.2A. A suspension system is first constructed using a spring device composed of three integrated flexible spiral arms, directly fabricated via 3D printing using PLA plastic as filament. A nitinol wire (`0.25 mm` diameter, `75 mm` length) is selected as the whisker shaft and attached to the top surface of the suspension device. A neodymium permanent magnet, axially magnetized with its field direction aligned with the wire, is attached at the bottom of the device. This upper assembly is supported by a 3D-printed structure positioned above a magnetic sensor (Mlx90393, Adafruit), which is configured to provide a high resolution of `0.15` $\mu T/LSB$ for measuring magnetic flux changes in three different directions. As a result, the contact reaction force from the object surface along the whisker shaft is converted into a motion of the magnet, leading to changes in the magnetic field detected by the Hall effect sensor. The magnetic sensor samples data at `300 Hz` and is connected to an Arduino Nano via I2C communication. Measurement data are transmitted to a computer at the same rate through USB serial communication, using a predefined ROS node on the Nano.

### Whisker Deflection Profile

The deflection profile of our whisker sensor consists of two primary elements: the root position, denoted as $\boldsymbol{p}_r$, and the measurement model, which maps the tangential contact location $\boldsymbol{c}_t$ to the corresponding deflection measurement $z$, as shown in Fig. 3B. Here, $\boldsymbol{c}_t = (x^b_t, y^b_t)$ represents the position vector in a 2D plane, with the superscript $b$ indicating the reference frame of the whisker sensor base. The deflection magnitude on the shaft is proven proportional to the bending rotation of the suspension system. We extract the flux change along the Y-axis of the magnetic sensor, which shows the most significant variation, to represent the deflection measurement $z$. Using a calibrated measurement model, a two-dimensional function $z_c = f(x^b, y^b)$ can be formulated to describe the deflection arc shape, where $z_c$ is the corresponding constant measurement value at given moment.

![Fig. 3](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig3-1.png){: .shadow}
_Fig. 3. (A) Setup for the 2-dimensional motorized stage to calibrate the whisker sensor. (B) Default root position $\boldsymbol{p}_r$ and extracted general deflection profile on whisker sensor, maps tangential contact point $(x^b_t, y^b_t)$ to measured magnetic flux change on Hall effector $z$. (C) Extract tip position by track the profile from tangent direction and loss direction based on gradient descent._

A 2-dimensional motorized stage is constructed for calibration, as shown in Fig. 3A. This stage consists of two orthogonal joints, with a touch rod attached vertically to the end-effector. The stage is driven by two NEMA17 stepper motors, and the motion of the end-effector is precisely measured using two rotary encoders (AS5600, AITRIP). Magnetic measurement data and tangential contact positions are recorded as the motors move in a grid-like pattern, with a fixed step of 3mm along both axes to touch the shaft. Data collection is limited to one side of the whisker sensor’s forward direction, ensuring a more accurate fit of our 5th-order bivariate polynomial model to the calibrated data. A total of 180 data sets are collected. Additionally, the origin of the whisker base frame is set as the default root position, $\boldsymbol{p}_r=(0, 0)$, though this does not precisely match the real situation. A linear displacement may occur at the center of rotation due to the current spring system design, which leaves gap for future improvement.

### Tip Contact Localization

**General Principle**: Given the default root position, the deflection profile, and a specified arc distance $L$ (the actual whisker shaft length), the tip position can be extracted by tracing from the start point along the known direction until the end of the trajectory.

We employ a constrained gradient descent method to trace the deflection profile. Initially, a loss direction is defined by minimizing the squared distance error from current deflection measurement to target, to maintain the constant measurement $z_c$. This ensures that the 3-dimensional measurement model from the previous calibration is constrained by the level set $f(x^b, y^b) = z_c$. To prevent the loss direction from merely guiding the trace onto the current deflection profile and getting stuck at a local minimum, a tangent direction is introduced to guide the tracing forward. Finally, by combining the loss and tangent directions, as illustrated in Fig. 3C, the trace advances in fixed step sizes (`1e-3 mm`), ensuring continuous progress along the deflection profile until the required total movement distance $L$ is completed.

To further reduce computation time, we calculate and resample 20 sets of tip position data to build a characterized model for use in real experiments. This approach enables us to determine the tip contact position within a 1ms duration. Polynomial regression of the calibrated measurement model yields a Root-Mean Squared Error (RMSE) of 0.231 and an R-squared value of 0.9944. The characterized model for tip position calculation achieves RMSE values of 0.0064 and 0.0088 for the X- and Y-axes, respectively, with an R-squared value very close to 1.0.

### Bayesian Filtering

Bayesian filtering is employed to further reduce fluctuation in tip contact localization caused by friction. A modified version of the Kalman Filter is implemented based on a **constant state assumption**: With the proposed active control strategy for the sensor's pose, the whisker shaft is maintained to collide with the immobile object at a fixed tip contact position in specific optimal deflection shape. Consequently, the prior prediction for the current step, $$\hat{\boldsymbol{x}}_k^-$$ ($\boldsymbol{x}$, state vector to represents the 2D tip position under whisker-base frame), remains unchanged and is equal to the estimate from the last step, $\hat{\boldsymbol{x}}_{k-1}$. The prediction and update process is listed below.

$$
\begin{equation}
\begin{array}{ll}
\mathit{Predict:} & \quad \mathit{Update:} 
\\ \hat{x}_k^- = \hat{x}_{k-1} & \quad K_k = P_k^- (P_k^- + R_k)^{-1} 
\\ P_k^- = P_{k-1} + Q & \quad \hat{x}_k = \hat{x}_k^- + K_k (z_k - \hat{x}_k^-) 
\\ & \quad P_k = (1 - K_k) P_k^-
\end{array}
\end{equation}
$$

The measurement noise $\boldsymbol{R}_k$ is updated empirically for each iteration based on the previous $N$ consecutive estimated tip positions from the characterized model. The extracted points and their covariance are used to represent the uncertainty. The process noise $\boldsymbol{Q}$ is fixed with covariance of $1e^{-5}\mathit{I}_2$, which is found to work well representing the confidence from active control. As a result, the standard deviation of the filtered results is reduced to `0.078` on the X-axis and `0.033` on the Y-axis.

### Active Motion Control

This active control policy makes an action that combines: a rotary transformation of the whisker sensor toward object surface according to a prediction of its local feature based on B-Spline curve, a normal displacement that actively approaches the edge based on current deflection measurement and a customized PID controller, and a tangential exploratory movement. 


<!-- \RestyleAlgo{ruled}
\SetKwComment{Comment}{// }{}
\begin{algorithm}[t!]
\small{
    \caption{\small\text{Whisker Active Perception}}\label{alg:three}
    \KwData{deflection measurement $z$, end-effector state $e$}
    \KwResult{control commands (target orientation $\theta$, linear velocity $v_x, v_y$)}
     initialize the reconstructed contacts deque $Q\gets \emptyset$ \;
     initialize the key points deque $Q_k\gets \emptyset$ \;

    \If{$z$ is received}{
        \If{$abs(z) \ge collisionThreshold$}{
            $contacted \gets 1$;
        }
        \If{$contacted$}{
            \Comment{\small{calculate tip position}}
            $Q.push(z2coordinates(z))$\;
            \If{$len(Q) = filterWindow$}{
                $updateMeasureNoise(Q)$\;
                $filter.predict(), filter.update(Q[-1])$\;
                $\boldsymbol{p}_{cur} \gets transform(filter.x, e)$\;
            }
            \If{$contactPoint$ is keypoint}{
                \Comment{\small{build BSpline curve}}
                $Q_k.push(\boldsymbol{p}_{cur}), c \gets bspline(Q_k)$\;
                $\boldsymbol{p}_{next} \gets c.extrapolate(u_{next})$\;
                $orient \gets arctan(slope(\boldsymbol{p}_{cur}, \boldsymbol{p}_{next}))$\;
            }
            $x_{vel} \gets controller.update(z, x_{vel})$\;
            $y_{vel} \gets constraint(x_{vel}, totalVelocity)$\;
        }
    }
    \Return $\left \{ orient, x_{vel}, y_{vel} \right \} $\;
    }

\end{algorithm} -->



#### a. Rotary Action:

![Fig. 4](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig4-1.png){: width="700" .shadow}
_Fig. 4. Prediction on next contact point using B-Spline curve and $n$ previous key reconstructs $$\left \{ \boldsymbol{p}_{k-i} \mid i=0, 1,...,n-1 \right \}$$. The slope between current contact point $\boldsymbol{p}_k$ and next prediction $$\hat{\boldsymbol{p}}_{k+1}$$ is regarded as the next orient direction._

The B-spline curve can construct continuous analytical model from known observation points and estimate the state of next non-observation points. The curve $S(\aleph)$ is represented as follows:
\begin{equation}
S(\aleph) = \sum_{j=0}^{n-1} c_j B_{j,k,t}(x)
\end{equation}
where $$\aleph\in \left \{ x,y \right \}$$ represents the 2D coordinates position in world-fixed frame, $c_j$ is the $j$-th spline coefficients and $n$ is the total number of the sampled control points. $B_{j,k,t}$ are B-spline basis functions of degree $k$ and knots $t$. 

Suppose the sensor is driven to move along the object contour and leaves reconstructed points based on contacts along its past trajectory, as shown in Fig. 4, the most recent $n$ points are used to construct the B-spline curve. However, the uncertainty from previous direct contact localization may cause fluctuations in local area, even after filtering. As a result, two consecutive reconstructions might overlap due to noise, leading to significant discontinuity in the B-spline's prediction. Therefore, these key points are extracted every $d$ iterations with an extra fixed interval distance, ensuring that they are orderly distributed and approximately evenly spaced (as the inferences are executed very fast with `300 Hz`, the contact points will not move far between iterations). In this scenario, the parameterization will be nearly uniform due to the even spacing, allowing for an extrapolated prediction when extending the parameter value beyond the original range as follows:

$$
\begin{equation}
\hat{\boldsymbol{p}}_{k+1} = f(u_{k+1}, \aleph_{k}, \aleph_{k-1}, \aleph_{k-2}, \ldots, \aleph_{k-n+1}) 
\end{equation}
$$

Where $u_{k+1}=1+1/(n-1)$ denotes the parameter value of the next key point, $$\left \{ \aleph_{k}, \aleph_{k-1}, \aleph_{k-2}, \ldots, \aleph_{k-n+1} \right \}$$ represents the $n$ selected previous key points. The algorithm is implemented based on SciPy, which by default uses centripetal parameterization. This method helps prevent clustering of parameter values, leading to better and more stable spline fitting. With the prediction of the next contact position $$\hat{\boldsymbol{p}}_{k+1}$$, we can calculate the slope from the current point, $s_{k\to k+1}$ and reconstruct the general direction of the local surface. The action then orients the sensor to the target angle, $arctan(s_{k\to k+1})$ according to the local curvature of the object, functioning as a tactile servoing mechanism.

#### b. Linear Displacement:
An optimal deflection target is empirically determined, where fluctuations are minimized and contact is consistently restricted to the tip of the shaft. After adjusting the orientation in the previous step, an active approaching movement is initiated to move the sensor radially toward the object, ensuring that the deflection magnitude remains at the defined optimal deflection state. The direction of this movement aligns with the sensor's orientation, and its derivative is automatically controlled by a customized proportional-integral-derivative (PID) controller.

A fixed total linear velocity is set to constrain the tangential exploratory movement in relation to the dynamics of the normal displacement. For example, if a drastic slope change is detected on the object's surface, the radial linear velocity will be adjusted accordingly, automatically slowing down the tangential pace along the edge while maintaining the predefined total velocity. This approach also supports the assumption of even spacing from the previous slope calculation. With optimal active control of the end-effector, the sensor should remain parallel to the object surface and maintain a consistent deflection shape, ensuring that a constant total linear displacement of the sensor corresponds to a consistent distance between tip reconstructions across iterations.

## V. Experiments

---

In this section, we first compare the localization performance across multiple trials on a flat surface to determine the optimal contact state. The proposed algorithm is then tested in simulations with an open wall trajectory. Finally, real-world tests are conducted on three different objects and their contours are reconstructed by millimeter-level accuracy.

### Optimal Contact State

The magnet's rotation and the corresponding measurement are proportional to the bending magnitude of the whisker shaft, which directly relates to the contact distance between the sensor and the object surface. To determine an optimal contact deflection target, we first compare the tip localization performance on a flat plane at different contact distances. In this experiment, a printed plane model is attached to the end-effector of our motorized stage. The platform is driven to move at a constant velocity along X-axis, colliding with the whisker shaft. The experiment consists of 9 trials, with data collected at different distances, each separated by a constant step distance of `5 mm`. Measurements are sampled at `300 Hz`, and the total moving distance for each trial is approximately 80mm.

The initial parameter settings for tip localization are consistent across all 9 trials. The algorithm begins calculating the tip position once a collision is detected, indicated by a deflection exceeding the threshold (set by default at 300 $\mu T$ above the original static measurement), and the corresponding tip position at this start moment is used as the initial prior mean for the Kalman filter. The initial prior covariance is set to $10.0\mathit{I}_2$, reflecting the uncertainty from the first contact. Given a calculation duration of less than `1 ms`, the inference is executed at the sensor sampling rate of `300 Hz`.

![Fig. 5](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig5-1.png){: width="700" .shadow}
_Fig. 5. (A) Results from 9 trials with varying contact distance are demonstrated. The mean absolute error comes from minimum distance between contact estimates and ground-truth trajectory, and corresponding distribution of groups are compared in the plot. (B) Magnetic flux change around Hall effect sensor is record during the test. The deflection on whisker is successfully maintained around its optimal state which corresponds to a magnetic measurement around 8760 $\mu T$. Two significant slips in the trajectory are also observed from the measurement which corresponds to the slips in octagon reconstruction from Fig. 7._

Results from trials are compared in Fig. 5A. In the experiments, our proposed localization method tracks the ground-truth contacts with a Euclidean distance error of less than `2 mm` across all nine trials. Notably, the trial at a height of `5 mm` achieves the most stable reconstruction, with a minimum standard deviation of 0.25. The trial at a height of `40 mm` yields the lowest average error of `0.08 mm`. These results demonstrate that our proposed localization method is capable of reconstructing tip contacts with relatively high accuracy, with the contact distances of `5 mm`, `30 mm`, `35 mm`, and `40 mm` tracking with sub-millimeter accuracy on average.

Finally, we select the bending state at `40 mm` as our primary deflection target. It provides the most accurate reconstruction on average, with relatively low variance. Importantly, it strikes a balance: it is neither too close to detaching from the object surface, as observed at `5 mm`, nor prone to slipping into tangential contact, as seen near `45 mm`. This choice leaves sufficient space for active control and increases the system's tolerance. The results from the first 8 trials show a smooth change in estimates, but a drastic decline near `45 mm`, followed by significant failure in subsequent trials, indicating that it had slipped into tangential contact.

### Reconstruction with Active Control

To validate the general effectiveness of our active tactile perception method, we first test it in a simulation environment. The simulated trajectory includes a flat surface, edges with gradually changing curvature, and sharp corners. The whisker body is modeled by connecting 40 elasticity cables in MuJoCo. As shown in Fig. 6, the reconstruction produced is promising. The sensor model is capable of continuously tracking the surface and navigating around the object without interruption.

![Fig. 6](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig6-1.png){: width="700" .shadow}
_Fig. 6. Reconstruction in simulation environment. A special trajectory is constructed to valid the general effectiveness of the method._

Finally, to further evaluate the performance, we collect data from contacts between the whisker sensor and three different objects in real world. The whisker sensor is mounted directly onto the end-effector of a robot arm (Franka Emika Panda), sampling data at `30 Hz`. The measurements are transmitted via a ROS-based serial node to the connected computer at the same frequency. The sensor starts to move at a fixed linear speed (`0.01 m/s`) and is brought into contacts with object by robot arm from random position. The calculation on active perception commence immediately after collision is detected. The scattered contour is reconstructed in real-time on a 2D plane, and active control commands are generated accordingly. The robot arm is simultaneously commanded to drive the sensor roaming around unknown objects using Cartesian velocity control on the end joint, continuing until the maximum rotary range of the joint is reached.

We select three different objects to test our proposed algorithm: a cylinder (diameter: `160 mm`), a rectangular prism with rounded corners (width: `160 mm`, radius: `40 mm`), and an octagonal prism (side length: `70 mm`, radius: `30 mm`). Knowing the basic shape and object location allows us to compare the estimated contact points on the object surface with the ground truth. The tracking results are shown in Fig. 7. Due to the rotary limit on the robot arm's end joint, we could not reconstruct the entire contour in one trajectory, resulting in a small section being missed. However, the available data is sufficient to demonstrate the effectiveness of our methods. 

Finally, average distance for each objects are: `0.547 mm` $\pm$ `0.371` for cylinder, `1.930 mm` $\pm$ `1.028` for rectangle, `0.842 mm` $\pm$ `0.741` for Octagon. 

![Fig. 7](/localdata/assets/EinsProject/Whisker-based%20Active%20Tactile%20Perception%20for%20Contour%20Reconstruction-Fig7-1.png){: width="700" .shadow}
_Fig. 7. Tracking the contact position and reconstruct the contours when making contact with different objects in real-world. Exemplary slips during the sweep and detailed numerical evaluation on reconstruction are given._

Results show that the object surfaces are accurately reconstructed from the contacts, producing distinguishable contour features on three unknown objects. Additionally, Fig. 5B illustrates the corresponding magnetic flux change in the selected direction of the magnetic sensor during the trial on the octagon. The algorithm is able to recover from a significant slip which occasionally happened in the trajectory. This further confirms that the system is capable of maintaining contact at our predefined optimal deflection state throughout the trajectory. The average bending rotation of the whisker sensor is maintained at `-8798.80` $\mu T$, slightly surpassing the target of `-8760` $\mu T$, which is reasonable since the sensor tends to detach from a convex surface. 

## VI. Discussion and Conclusion

---

In this paper, we demonstrate the potential for a reliable active tactile perception using a whisker-inspired sensor. The suspension device is further compacted for our magnetically transduced whisker sensor by introducing three integrated spiral arms. By integrating a tip contact localization with active motion control policy, the sensor is enabled to continuously follow the unknown surface with its optimal deflection state, and reconstruct the contacts in sub-millimeter accuracy. In the future, we aim to further improve our method from: 
- incorporate tangential contact localization into the system to enhance the robustness of active tactile reconstruction on surfaces with varying curvatures, where axial slipping may occur; 
- integrate multiple whiskers into a sensory array and develop an optimization algorithm based on their morphological patterns to improve reconstruction accuracy.
