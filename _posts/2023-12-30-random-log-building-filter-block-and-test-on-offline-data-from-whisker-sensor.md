---
layout: post
title: Building Filter Block and Test on Offline Data from Whisker Sensor
date: 2023-12-30 22:19 +0100

categories: [Eins-Project, Random Log]
tags: [Whisker, State Estimate]

math: true
mermaid: true

post_description: Build process model based on static contect hypothesis and inverse uniquly mapping measurement according to the previous studies, moreover, make a test on offline data from whisker sensor.  
---

## Problem Statement

---

2D contact position inference for whisker sensor. The basic format of state estimation equation is showed as follows:

$$
x_{k+1}=\mathbf{A}x_{k}+\mathbf{B}\begin{bmatrix}v_{sb}^{b}\\w_{sb}^{b}\end{bmatrix}+w_{k} \\ y_{k}=g(x_{k})+v_{k}
$$

The problem was under 2D scene therefore the contact point was descripted by a 2-dimensional vector and the most important hypothesis made here was that, the contact was always static that is to say the position was fixed under world reference frame. 

## Process Model

---

Back to problem description, $v_{sb}^b$ and $w_{sb}^b$ are the linear and angular velocity respectively, of sensor base $$\left \{ B \right \}$$  relative to world-fixed frame $$\left \{ S \right \}$$  as viewed in the body frame. $x_k=p_c$ is the position vector of the contact point relative to origin of $$\left \{ B \right \}$$  at time-step $k$. We can first calculate the linear velocity of $p_c$ with **respected to world frame**:

$$
v_{sp_c}^b=v_{sb_p}^b+v_{bp_c}^b=v_{sb}^b+w_{sb}^b\times b_p+v_{bp_c}^b
$$

Define a point $b_p$ that is instantaneously coincident to $p_c$ but fixed in  $$\left \{ B \right \}$$ . The movement of contact under world frame was actually divided into two part: 1) the constant whisker base movement which correspond to the movement of  fixed point $b_p$ 2) and the contact shift under base frame $v_{bp_c}^b$ after a nominal short time interval $dt$.

![Untitled](/localdata/assets/EinsProject/WhiskerDeflectionStaticContactHypo.png)

Given the hypothesis we made at the very begin, the contact was always static in the world frame, thus there’s $v_{sp_c}^b=0$ ( or say $v_{bp_c}^b=-v_{sb_p}^b$, there’s another perspective to under this problem, we can conclude that the moving distance for $b_p$ in $$\left \{ S \right \}$$ equals to it for $p_c$ in $$\left \{ B \right \}$$ within a nominal short time deviation $dt$).  And we can calculate the $v_{bp_c}^b$ as follows:

$$
\begin{equation}
v_{bp_c}^b=-v_{sb}^b+-w_{sb}^b\times b_p = -\begin{bmatrix}
 \mathbf{I} & -\left [ \mathbf{p_c} \right ] 
\end{bmatrix}\begin{bmatrix}
v_{sb}^b \\ w_{sb}^b
\end{bmatrix}
\end{equation}
\tag{1}
$$

The process model can expressed as

$$
x_{k+1}=x_k+\delta_tv_{bp_c}^b=x_k-\delta_t \begin{bmatrix}
 \mathbf{I} & -\left [ \mathbf{p_c} \right ] 
\end{bmatrix}\xi_{sb}^b+w_k
$$

which gives $\mathbf{A}=\mathbf{I}$ and $\mathbf{B}=-\delta_t \begin{bmatrix}
 \mathbf{I} & \left [ \mathbf{p_c} \right ] 
\end{bmatrix}$.

<aside>
❓ It is of vital importance to understand what exactly happened in equation (1), yet there seems to be some mistake of the formulation in original paper. I’m not totally sure. There’s should be a minus before the skew-symmetric matrix of $p_c$, to set the cross product in order:

$$
-w_{sb}^b \times b_p = b_p \times w_{sb}^b = \left [ p_c \right ]w_{sb}^b
$$

But there’s none in the paper, I have to make sure who’s right here…

</aside>

To implement it as a state transition function in Kalman Filter:

```python
v_bpc_b = -v_sb_b + -np.cross(w_sb_b, bp ) # direct corss product 

def f_contact(x, lv, av, dt):
    input = np.append(lv, av, axis=0)
    B = -dt*np.append(np.identity(3), -skew(x), axis=1)
    A = np.identity(3)
    prior_predict = A @ x + B @ input 
    return prior_predict
```

The function will return prior prediction without process noise for later operation. 

BTW, we could calculate the contact point velocity $v_{bp_c}^b$ either by direct cross product according to the front part of (1) or in a totally matrix multiplication manner which could save the computation cost. 

## Measurement Model

---

## Other Details

---

- **Contact along surface - dynamic contact solution**
    
    Back to the hypothesis we have made at very begin, “the contact was always static”. Presume there’re two types of contact in the real world: 1) contact by a rigid touch rod; 2) contact by real object along its surface and whisker length. In the former situation, it is easy to imagine that the contact position, which attached on the rigid rod, is always fixed in the world frame. Of course it is since the rod would never move, only the whisker itself. However as for the latter situation, the contact will continue to shift along the surface with a dynamic position in world frame. Therefore the process model would fail here. 
    
    To compensate that, with no applicable modeling for that part of state transition, we can either 1) introduce an extra fictitious process noise in the model; 2) or conduct fading memory (FM) in the calculation of estimation error covariance:
    
    $$
    \tilde{\mathbf{P}} = \alpha^2\mathbf{FPF^\top}+\mathbf{Q}
    $$
    
    Neither of these two approach bear a great confidence on prior prediction thus decrease its proportion on the data fusion in different manner. By conduct a scale factor $\alpha$, thus it spare greater weight on the more recent measurements, and forget the history estimation (that is to say, “fading memory”). 
    
    ```python
    filter.Q = Q_discrete_white_noise(dim=2, dt=dt, var=20.) # increase process noise
    filter.alpha = 1.00 # scaling factor of FM
    ```
    
    Both of which were built in the implementation of Kalman Filter in filterpy. 
    
- **Inertial measurement and offline data collection**