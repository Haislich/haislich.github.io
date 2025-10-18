---
title: "Myocontrolled neuroprosthesis integrated with a passive esoskeleton to support upper limb activities"
permalink: /master-thesis/ambrosini-et-al/
usemathjax: true
# last_modified_at: 2021-06-07T08:48:05-04:00
redirect_from:
  - /theme-setup/
toc: true
---

## Introduction

Ambrosini et al. propose to use neuromuscular electrical stimulation (NMES), also known as functional electrical stimulation (FES), to assist and rehabilitate people with impaired motor functions. The main motivation is that NMES uses the patient’s own muscles as actuators, which promotes neuroplasticity and prevents muscle atrophy, unlike orthoses that provide external support. By synchronizing the stimulation with the user’s voluntary effort, it is possible to both aid movement execution and reinforce the neural drive associated with that movement.

The problem lies in the hybrid nature of muscle activation when NMES is applied. The total EMG recorded from a stimulated muscle can be expressed as

$$
EMG_{total} = EMG_{volitional} + EMG_{stimulation}
$$

where ( EMG_{volitional} ) is the natural electrical activity produced by voluntary contraction and ( EMG_{stimulation} ) is the component induced by electrical stimulation. The stimulation part includes the stimulation artifact (a high-amplitude, short-duration spike) and the M-wave, a compound action potential resulting from synchronous activation of fibers. The M-wave is typically one to two orders of magnitude larger than ( EMG_{volitional} ), and since its shape and amplitude vary slowly over time, it contaminates the signal of interest. This time variance, called nonstationarity, arises from both physiological and observational sources. Physiological causes include muscle fatigue and changes in recruitment pattern, while observational causes include electrode shift, variations in skin–electrode impedance, and changes in muscle fiber orientation during movement. The result is that ( EMG_{total} ) cannot be directly used to estimate ( EMG_{volitional} ).

Even if ( EMG_{volitional} ) could be estimated perfectly, the control problem remains complex. The system must translate the estimated neural drive into stimulation parameters that produce appropriate muscle torque. Two main control strategies have been used. The first is trigger-based control, where stimulation is fired once ( EMG_{volitional} ) exceeds a fixed threshold. This method is robust but crude, since the stimulation intensity is predetermined and cannot adapt to the user’s effort. The second approach is proportional control, where the stimulation intensity is made proportional to the estimated voluntary EMG:

[
Stimulation = k \cdot EMG_{volitional}
]

with ( k ) a proportionality constant. This theoretically synchronizes the artificial and natural contractions, but it often leads to unstable behavior. When the user increases effort to move, EMG rises, the controller increases stimulation, and total torque overshoots the intended value. The user then relaxes, EMG falls, stimulation decreases, and the motion undershoots. The alternation between overshooting and relaxation causes jerky oscillations. To reduce high-frequency noise in ( EMG_{volitional} ), low-pass filtering is typically applied, but this introduces a time delay ( \Delta t ), which means the system reacts to outdated information. The combination of proportional response and delay produces closed-loop instability. Both trigger-based and proportional controllers therefore feel unnatural to use, motivating Ambrosini’s search for a more stable and intuitive control strategy.
