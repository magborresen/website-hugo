---
categories: matlab
date: "2023-06-23"
description: Implementation of a Permanent Magnet Synchronous Motor in MATLAB SimSpace
tags: ["matlab", "pmsm", "simscape", "simulation"]
author: ["me"]
title: PMSMs in SimScape
math: true
draft: true
---

One of the major parts of my thesis was developing some sort of setup to simulate faults in electric motors.

After some initial research, I chose to work on inter-turn faults for Permanent Magnet Synchronous Motors (PMSM) and Brushless DC (BLDC) motors. The inter-turn faults occur in the windings of the three motor phases (yes, I was working on three-phase motors as well). When the motor is driven over its lifetime either within or outside of its specification, wear begins to deteriorate the motor. Some of the first signs in the electrical domain are often inter-turn faults.

I have included a figure of the most common electrical faults in BLDC and PMSMs that you can see below.

{{< figure src="stator_faults.png" >}}

Inter-turn faults are shorts between windings in one of the coils making up one or more phases of the motor. During the motor's lifetime, the enameled coating on the winding starts to wear away. Once it has worn away on two adjacent windings, a short has occurred. This short will change the characteristics of the motor. If unnoticed, the windings might continue to deteriorate causing even more problems.

Several different methods for detection has been proposed. Many of these are summarized in this review [^1].

Due to time constraints on the project, it was unfortunately not possible to develop a full-scale experimental setup.

Instead, I chose to work on a simulation model. Since I was working with Infineon for the thesis, I was hoping they already had a model ready that I could use. I quickly found out this was not the case. So I was stuck again. I had to develop one from scratch. Mathematical models for PMSMs are very well defined in litterature. But there are slight differenced in many of the definitions. Mathematical models are also not easily implementable for simulation. One could implement something using Simulink blocks, but this usually only works well for one scenario.

We wanted something that would work well in all the scenarios we could throw at it. The only way to accomplish this is MATLAB Simulink, is to implement the model in SimScape. 

SimScape is a language that builds on top of Simulink and the models created within the language can be freely used in Simulink. It is more or less a mathematical languague with some alterations. There are input and output definitions along with parameters that must be taken care of. MATLAB already has an implementation of a PMSM with thermal parameters. But unfortunately it is a closed source model, so no alterations to introduce faults can be made. 

Meaning we needed to create a model from scratch. So we first need to define the mathematics that will be used for implementation. There are different opinions in litterature as to how such a model should be defined. I will show you the method that I chose and explain the differenced along the way. 

# Healthy Motor Model

The electrical equivalent model of a healthy (right) and faulted (left) PMSM is shown in the figure below

{{< figure src="fault_electrical_model.PNG" title="Electrical model of healthy (right) and faulted (left) PMSM" >}}

This model has been implemented based on the equations and methods described in [^2].

Next, we can set up the defining equations for such a model for the healthy case of the motor. As the motor has three phases that is each driven using a supply voltage, the equation for phase voltages can be given as 

$$ v_{abc} = R_{abc} i_{abc} + \frac{dL_{abc} i_{abc}}{dt} + e_{abc} $$

With $R_{abc}$ being the resistance matrix of the series resistances in each of the motor phases, making it a $3\times 3$ matrix. 

$$ R_{abc} = \begin{bmatrix} 
                R_s & 0 & 0 \\\\
                0 & R_s & 0 \\\\
                0 & 0 & R_s 
            \end{bmatrix} $$

The phase currents $i_{abc}$ are defined as a $3 \times 1$ vector

$$ i_{abc} = \begin{bmatrix}
                i_a \\\\
                i_b \\\\
                i_c
            \end{bmatrix} $$

The inductance matrix for the series inductances of each phase is a little different. This is where most of the litterature doesn't agree any longer. Some researchers believe that the motor should be modelled with inductances that fluctuate according to rotor angle. Other researchers argue that constant inductances should be used.

The mutual inductances are also up for debate. The series inductances creates the diagonal of the $3 \times 3$ matrix. However all of the other entries are mutual inductances between the phases. The debate is whether the mutual inductances matter. I have chosen to include them.

$$ L_{abc} = \begin{bmatrix}
                L_{aa} & L_{ab} & L_{ac} \\\\
                L_{ba} & L_{bb} & L_{bc} \\\\
                L_{ca} & L_{cb} & L_{cc}
            \end{bmatrix}
$$

The matrix is symmetrical because the mutual inductances are equal for example for $L_{ab}$ and $L_{ba}$. Since we assume that the motor is symmetric and balanced, all the series inductances are equal. However, since the angle reference is at phase A, the angles of phase B and C must be shifted

$$L_{aa} = L_s + L_m - L_{\Delta}cos(2\theta_e)$$
$$L_{bb} = L_s + L_m - L_{\Delta}cos(2(\theta_e - 2\pi / 3))$$
$$L_{cc} = L_s + L_m - L_{\Delta}cos(2(\theta_e + 2\pi / 3))$$
$$L_{ab} = -0.5L_m - L_{\Delta}cos(2(\theta_e + \pi/6))$$
$$L_{ac} = -0.5L_m - L_{\Delta}cos(2(\theta_e + \pi/6) + 2\pi / 3)$$
$$L_{bc} = -0.5L_m - L_{\Delta}cos(2(\theta_e + \pi/6) - 2\pi / 3)$$

With $L_s$ being the series inductances of the motor phases, $L_m$ is the mutual inductance between the windings and $L_{\Delta}$ is the inductance fluctuation coefficient. $\theta_e$ is the electrical rotor angle.

Lastly, we must define the back-EMF, which is the $e_{abc}$ from the voltage equation. The back-EMF is a voltage source within the motor and for a PMSM it will produce a sinusoidal voltage.

$$e_{abc} = \omega \psi_m \begin{bmatrix} cos\theta_e \\\\ 
                                        cos\theta_e + 2\pi / 3 \\\\ 
                                        cos\theta_e - 2\pi / 3 \end{bmatrix} $$

Here, $\omega$ is the speed of the rotor, $\psi_m$ is the permanent magnetic flux of the motor. 

As you can see from the equations, everything is referenced to phase A (U) of the motor. The calculations for the two other phases are shifted in the cosine expressions.

Lastly, we must define the output of the motor, which will be the electromagnetic torque. This is the torque seen at the electrical side of the rotor shaft of the motor.

$$ T_E = P_P\left(\frac{1}{2}i^T_{abc} \frac{dL_{abc}}{d\theta_e} + \frac{i_{abc}^T e_{abc}}{\omega} \right) $$

Again we've got a few new variables to take care of. So $P_P$ is the number of pole pairs in the motor. This number can fluctuate a lot between the motor models. The source of these equations points to that they are only valid for $P_P=1$. The second new variable is $\omega$, which represents the speed of the rotor. When implementing in SimScape, it is possible to get the angle output of the motor, where the speed is the derivative of the angle, as shown by classical mechanics. 

Now we basically have the mathematical description of a healthy motor, let's then look at what it takes to alter these equations to reprensented a motor with various levels of inter-turn faults.

# Faulted Motor Model

To implement a faulted model as the one shown in the previous image, it is necessary to make a few altercations to the equations that I described before. What happens when the coating between the individual windings starts to wear away, is that another coil is created due to the short circuit between the windings. This new coil will have its own charactestics just as the normal coil. We must therefore define a resistance and inductance for this new coil, along with adding the effect of the extra back-EMF produced. 

Practically, this means expanding the previous matrices with another row and column.

What we are expanding with, is a multiple of the ratio between healthy and faulted winding in the motor. This value is defined by $\sigma = \frac{N_f}{N_{tot}}$. Meaning the ratio between the faulted and total number of windings in the coil. 

We can then rewrite the resistance matrix into

$$ R_{abcf} = \begin{bmatrix} 
                R_s & 0 & 0 & 0 \\\\
                0 & (1-\sigma)R_s & 0 & 0 \\\\
                0 & 0 & R_s & 0 \\\\
                0 & 0 & 0 & \sigma R_s
            \end{bmatrix} $$

The matrix shows that a fourth column and row has been added with the resistance value of the new faulted coil. The resistances in the faulted B phase is reduced linearly with to the remainder of the otherwise healthy turns.

Having done this, we must also add an extra row to the current vector since we will now have a circulating current in phase B.

$$ i_{abcf} = \begin{bmatrix}
                i_a \\\\
                i_b \\\\
                i_c \\\\
                i_f
            \end{bmatrix} $$

The newly created faulted coil also affects the inductance matrix. So like we did in the resistance matrix, we must add another row and column in order to accomodate. Changes must be made to all inductances that are influenced by phase B. Thus not only the series inductances but the mutual inductances as well.

$$ L_{abcf} = \begin{bmatrix}
                L_{aa} & L_{ab} & L_{ac} & \sigma L_{ab} \\\\
                (1-\sigma)L_{ba} & (1-\sigma)^2L_{bb} & (1-\sigma)L_{bc} & (1-\sigma)\sigma L_{bb} \\\\
                L_{ca} & (1-\sigma)L_{cb} & L_{cc} & \sigma L_{cb} \\\\
                \sigma L_{ba} & (1-\sigma)\sigma L_{bb} & \sigma L_{bc} & \sigma^2 L_{bb} 
            \end{bmatrix}
$$

From this, it can easily be seen that not all the elements of the matrix are scaled linarly with the changes to $\sigma$. My experience is that this unfortunately leaves the matrix being ill-conditioned. The ill-conditioning makes it hard for the system to converge in complex system setups within Simulink. However this does work in less complex simulation models.

Lastly, we must modify the vector that contains the expressions for the back-EMF. Since we have now added a new coil to the model, there will be an additional unit of back-EMF. The expression for the back-EMF of phase B will also change accordingly.

$$e_{abcf} = \omega \psi_m \begin{bmatrix} cos(\theta_e) \\\\ 
                                        (1-\sigma)cos(\theta_e - \frac{2\pi}{3}) \\\\ 
                                        cos(\theta_e + \frac{2\pi}{3}) \\\\
                                        \sigma cos(\theta_e - \frac{2\pi}{3})
                                        \end{bmatrix} 
                                        $$

Now that we have redefined all of our matrices and vectors, it is time to write up the new expression for calculating the voltage in each phase. This will be a vector calculation as we will get a voltage in phase A, B, C and the faulted phase called F.

$$ v_{abcf} = R_{abcf} i_{abcf} + L_{abc} \frac{di_{abc}}{dt} + i_{abcf} \frac{dL_{abcf}}{dt} + e_{abc} $$

As you can see, we have some expressions of the inductance matrix and currents that require derivatives. The derivative of the current can be found using SimScape when the model has been implemented. But the time derivative of the inductance matrix must be defined now. This is easily done though.

$$\frac{dL_{aa}}{dt} = 2\omega \cdot L_{\Delta}cos(2\theta_e)$$
$$\frac{dL_{bb}}{dt} = 2\omega \cdot L_{\Delta}cos(2(\theta_e - 2\pi / 3))$$
$$\frac{dL_{cc}}{dt} = 2\omega \cdot L_{\Delta}cos(2(\theta_e + 2\pi / 3))$$
$$\frac{dL_{ab}}{dt} = 2\omega \cdot L_{\Delta}cos(2(\theta_e + \pi/6))$$
$$\frac{dL_{ac}}{dt} = 2\omega \cdot L_{\Delta}cos(2(\theta_e + \pi/6) + 2\pi / 3)$$
$$\frac{dL_{bc}}{dt} = 2\omega \cdot L_{\Delta}cos(2(\theta_e + \pi/6) - 2\pi / 3)$$

Now we need to redefine how the torque is produced. We've previously set up an equation for the electromechanical torque seen at the rotor shaft. But now we need to take our now values into account. 

$$ T_E = P_P\left(\frac{1}{2}i^T_{abcf} \frac{dL_{abcf}}{d\theta_e} + \frac{i_{abcf}^T e_{abcf}}{\omega} \right) $$

With the angular derivative of the inductance matrix being defined as

$$\frac{dL_{aa}}{dt} = 2 \cdot L_{\Delta}cos(2\theta_e)$$
$$\frac{dL_{bb}}{dt} = 2 \cdot L_{\Delta}cos(2(\theta_e - 2\pi / 3))$$
$$\frac{dL_{cc}}{dt} = 2 \cdot L_{\Delta}cos(2(\theta_e + 2\pi / 3))$$
$$\frac{dL_{ab}}{dt} = 2 \cdot L_{\Delta}cos(2(\theta_e + \pi/6))$$
$$\frac{dL_{ac}}{dt} = 2 \cdot L_{\Delta}cos(2(\theta_e + \pi/6) + 2\pi / 3)$$
$$\frac{dL_{bc}}{dt} = 2 \cdot L_{\Delta}cos(2(\theta_e + \pi/6) - 2\pi / 3)$$

Now we've got all of our values in check. We can then start looking at how we implement this into a SimScape model. 

# SimScape Model Implementation

[//]: # (Source Section) 

[^1]: [Review of Methods for Diagnosing Fault in the Stators of BLDC Motors](URL "https://www.mdpi.com/2227-9717/11/1/82")
[^2]: [Interior Permanent Magnet Synchronous Motor Stator Winding Fault Modelling](URL "https://www.sciencedirect.com/science/article/pii/S2405896315008307")