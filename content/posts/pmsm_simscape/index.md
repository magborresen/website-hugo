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

Inter-turn faults are shorts between windings in one of the coils making up one or more phases of the motor. During the motor's lifetime, the enameled coating on the winding starts to wear away. Once it has worn away on two adjacent windings, a short has occured. This short will change the characteristics of the motor. If unnoticed, the windings might continue to deteriorate causing even more problems. 

Due to time constraints on the project, it was unfortunately not possible to develop a full-scale experimental setup.

Instead, I chose to work on a simulation model