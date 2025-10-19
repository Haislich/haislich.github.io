---
title: "Myocontrolled neuroprosthesis integrated with a passive esoskeleton to support upper limb activities"
permalink: /master-thesis/ambrosini-et-al/
usemathjax: true
redirect_from:
  - /theme-setup/
toc: true
---

## Introduction

Ambrosini et al. propose to use neuromuscular electrical stimulation (NMES), also known as functional electrical stimulation (FES), to assist and rehabilitate people with impaired motor functions. The main motivation is that NMES uses the patient’s own muscles as actuators, which promotes neuroplasticity and prevents muscle atrophy, unlike orthoses that provide external support. By synchronizing the stimulation with the user’s voluntary effort, it is possible to both aid movement execution and reinforce the neural drive associated with that movement.

### The sensing problem

The problem lies in the hybrid nature of muscle activation when NMES is applied. The total EMG recorded from a stimulated muscle can be expressed as

$$
EMG_{total} = EMG_{volitional} + EMG_{stimulation}
$$

where $EMG_{volitional}$ is the natural electrical activity produced by voluntary contraction and $EMG_{stimulation}$ is the component induced by electrical stimulation.
The stimulation part includes the stimulation artifact (a high-amplitude, short-duration spike) and the M-wave, a compound action potential resulting from synchronous activation of fibers.

|![M wave and stimulus](https://www.researchgate.net/profile/Yuan-Yang-32/publication/328964432/figure/fig5/AS:757401594654720@1557590020218/Combination-of-M-wave-and-stimulation-artifact-for-creating-semi-synthetic-signals-for.ppm)|
|---|
|Combination of M-wave and stimulation artifact|

The M-wave is typically one to two orders of magnitude larger than $EMG_{volitional}$, and since its shape and amplitude vary slowly over time, it contaminates the signal of interest.
This time variance, called nonstationarity, arises from both physiological and observational sources.
Physiological causes include muscle fatigue and changes in recruitment pattern, while observational causes include electrode shift, variations in skin–electrode impedance, and changes in muscle fiber orientation during movement.
The result is that $EMG_{total}$ cannot be directly used to estimate $EMG_{volitional}$.

### The control problem

Even if $EMG_{volitional}$ could be estimated perfectly, the control problem remains complex.
The system must translate the estimated neural drive into stimulation parameters that produce appropriate muscle torque. Two main control strategies have been used.
The first is trigger-based control, where stimulation is fired once $EMG_{volitional}$ exceeds a fixed threshold.
This method is robust but crude, since the stimulation intensity is predetermined and cannot adapt to the user’s effort. The second approach is proportional control, where the stimulation intensity is made proportional to the estimated voluntary EMG:

$$
Stimulation = k \cdot EMG_{volitional}
$$

with $k$ a proportionality constant.
This theoretically synchronizes the artificial and natural contractions, but it often leads to unstable behavior. When the user increases effort to move, EMG rises, the controller increases stimulation, and total torque overshoots the intended value.
The user then relaxes, EMG falls, stimulation decreases, and the motion undershoots.
The alternation between overshooting and relaxation causes jerky oscillations. To reduce high-frequency noise in $EMG_{volitional}$, low-pass filtering is typically applied, but this introduces a time delay $\Delta t$, which means the system reacts to outdated information.
The combination of proportional response and delay produces closed-loop instability.
Both trigger-based and proportional controllers therefore feel unnatural to use, motivating Ambrosini’s search for a more stable and intuitive control strategy.

## Phase A

Ambrosini distinguishes between two fundamental problems in EMG-driven stimulation:

1. extracting the voluntary component $EMG_v$ from the hybrid EMG that also contains the stimulation-induced M-wave,
2. using that cleaned signal for stable control of the stimulator.

To address them separately, the work is divided into two phases. **Phase A** focuses on understanding and filtering the hybrid EMG in an **offline** setting; **Phase B** builds on this to design a real-time myocontrolled stimulator.

The need for an offline study comes from the fact that before closing the control loop on a human subject, we must first understand what the raw signals look like and how stimulation distorts them. This phase provides that groundwork.

Ten healthy subjects and eight neurological patients participated.
Healthy subjects were included because their muscles are reliable and fatigue slowly; their responses provide a consistent reference for identifying how a “clean” EMG behaves under stimulation. Knowing that reference helps interpret the noisier patient data.

### Experimental protocol

The stimulation was applied to the biceps brachii with these parameters:

- Pulse width (duration of each phase): $\approx 300 \mu s$
- Current amplitudes: two levels $C_1>C_2$, producing stronger and weaker contractions. Different currents produce different M-wave amplitudes, so both were tested.
- Frequency: fixed at 25 Hz (one pulse every 40 ms).

The 25 Hz frequency is a practical trade-off.
Lower frequencies would give longer stimulation periods and therefore more “clean” EMG between pulses, but the muscle response would be twitchy and weak.
Higher frequencies would produce smooth tetanic contractions but leave very little usable EMG between pulses.
At 25 Hz, contraction is nearly fused while a short usable EMG window (~20 ms) remains after each pulse.

Each trial contained five 5- to 10-s phases designed to create known, reproducible conditions:

0. **Rest (10 s):** no stimulation, no voluntary effort → baseline noise.
1. **$C_1$ stimulation, no effort (5 s):** purely evoked M-wave at high amplitude.
2. **$C_2$ stimulation, no effort (5 s):** same as (1) but weaker current, hence smaller M-wave.
3. **$C_2$ stimulation + voluntary effort (5 s):** hybrid case (M-wave + vEMG).
4. **Dynamic $C_1$:** triangular current profile to observe a time-varying M-wave.

### Pre-processing

Immediately after each pulse, the amplifier saturates due to the electrical artifact.
Those first samples are removed using a **blanking window**:

- 20 ms for the adaptive filter (retain more of the M-wave).
- 27 ms for the high-pass filter (discard as much M-wave as possible).

After blanking, only the remaining part of each 40 ms stimulation period is usable.
The EMG is then **rectified** so that depolarization and hyperpolarization contribute equally to signal magnitude.

A schematic period therefore looks like:

<pre class="mermaid">
packet
0-9: "Blanked EMG"
10-19: "Usable EMG"
</pre>

The software segments the signal into these periods of (N) samples $(N=f_s/f_{stim}=2048/25≈82)$, zeros or removes the blanked region, and rejoins the rest into a continuous sequence $EMG_f(n)$.

### Adaptive filter

$$
EMG_v(n)=EMG_f(n)-\sum_{j=1}^{M} b_j*EMG_f(n-jN)
$$

where

- $EMG_f(n)$: pre-processed EMG (blanked + filtered), indexed globally $n=0\dots L-1$,
- $EMG_v(n)$: estimated voluntary EMG,
- $N$: number of samples per stimulation period $\approx 82$,
- $M$: number of preceding periods used for prediction $\approx 6$,
- $b_j$: linear-prediction weights, estimated once per period by least squares using only **unblanked** samples.

Least-squares estimation:

$$
\underset{\min} {b_1, \dots,b_M}
\Big[EMG_f(n)-\sum_{j=1}^{M}b_j*EMG_f(n-jN)\Big]^2,
$$

where $\mathcal{S}_k$ = set of unblanked indices in period $k$.

This filter relies on two assumptions:

1. The **M-wave** is **slowly time-variant** and **periodic** across stimulation cycles. Its shape drifts slowly due to fatigue, electrode impedance, or geometry changes.
2. The **volitional EMG** is **band-limited** $\approx 30–300 Hz$ and **statistically random** (zero-mean Gaussian amplitude distribution, uncorrelated between periods).

Hence:

$$
EMG_f(n) = M(n) + v(n)
$$

The adaptive predictor models what is **correlated** across periods (the M-wave) and ignores what is uncorrelated (the voluntary part).
Subtracting the prediction removes the stimulation-locked component:

$$
EMG_v(n)=EMG_f(n)-\hat{M}(n)
$$

This yields a clean estimate of the volitional EMG without explicitly computing the M-wave itself.
However, for the filter to learn these correlations, the **M-wave must be visible** in the recorded EMG; if hardware blanking hides it, the method fails, this is the practical limitation Ambrosini notes in the discussion.

### High-pass filter

A simpler alternative uses a 2nd-order non-causal Butterworth high-pass filter with a 200 Hz cutoff.
Because this method does not model the M-wave, the blanking window is extended to 27 ms to let most of it decay.
The assumption is that the M-wave’s energy lies below 200 Hz while voluntary EMG dominates above 200 Hz.
In reality the spectra overlap: some M-wave energy leaks through and some voluntary energy is lost, leading to residual contamination and amplitude underestimation.

### Outcome of Phase A

The adaptive filter outperforms the high-pass filter: it provides near-zero output when only stimulation is present and accurately tracks voluntary effort during hybrid activation.
This demonstrates that the adaptive approach can reliably extract the volitional component, providing the foundation for the real-time controller developed in Phase B.

## Phase B

After Phase A identified the **adaptive filter** as the most effective method for extracting the voluntary component of the EMG, Phase B used it to drive a **real-time myocontrolled neuroprosthesis**.
The goal was to let subjects **activate and deactivate** electrical stimulation support for elbow flexion using their own filtered EMG signal.

This control strategy is **intermediate** between EMG-triggered and EMG-proportional methods.
The stimulation frequency remains constant at 25 Hz, while the **pulse width (PW)** is modulated smoothly according to the filtered and rectified EMG amplitude.

When the estimated voluntary EMG $EMG_v$ crosses a preset activation threshold $e_{ON}$, the controller increases PW linearly with slope $K$ until a maximum value $PW_{max}$ is reached.
When $EMG_v$ falls below a deactivation threshold $e_{OFF}$, the controller symmetrically decreases PW down to zero.
Between $e_{OFF}$ and $e_{ON}$ the PW is held constant.
Mathematically:

$$
\dot{PW} =
\begin{cases}
+K, & EMG_v > e_{ON} \text{ and } PW < PW_{max} \\
-K, & EMG_v < e_{OFF} \text{ and } PW > 0 \\
0, & e_{OFF} \le EMG_v \le e_{ON}
\end{cases}
$$

This hysteresis band $(e_{OFF}, e_{ON})$ prevents rapid switching when EMG fluctuates near the threshold, reducing oscillations and making stimulation feel smoother and more natural.

### Experimental setup

To minimize unwanted movement due to gravity or fatigue, subjects’ arms were supported by a **passive exoskeleton**.
This mechanical stabilization isolates the effect of the stimulation, allowing cleaner observation of controller behavior.
However, it also means that EMG recorded in this setup is somewhat noisier than in free motion, because of mechanical contact and friction in the exoskeleton.

Healthy subjects were used even though they could already perform the movement unaided.
For them, stimulation does not provide functional benefit but serves as a **validation environment** to evaluate controller stability and responsiveness under consistent conditions.
Because they can complete the movement regardless of assistance, we can directly compare their performance **with and without** stimulation and observe how the controller modulates support in real time.

### Performance evaluation

Subjects were asked to follow a **trapezoidal reference trajectory** of elbow flexion.
Performance was quantified using:

1. Root Mean Square Error (RMSE) between the target angle and the actual elbow angle, measuring movement accuracy.
2. Integrated voluntary EMG the time integral of $\|EMG_v\|$, measuring muscular effort.

These two quantities together indicate whether stimulation truly assists movement.
If RMSE stays constant or decreases while the integrated EMG decreases, it means that stimulation is effectively sharing the load.

### Observations

This on/off adaptive controller successfully removes the jerky and unnatural behavior typical of proportional schemes.
The slow ramping of PW and hysteresis thresholds make the interaction stable and comfortable.
However, the system requires **manual calibration** of thresholds $e_{ON}$ and $e_{OFF}$ for each subject and session to match their resting EMG level and contraction capacity.
This need for calibration remains one of the practical limitations of the method.

## Conclusions

Ambrosini’s overall conclusion is that a **myocontrolled neuroprosthesis** combining NMES with a passive exoskeleton can provide smooth, intuitive, and reliable upper-limb assistance when driven by a properly filtered EMG signal. The study demonstrates that an **adaptive linear prediction filter** can successfully separate the voluntary and stimulation-evoked components of a hybrid EMG, enabling real-time control without the instability typical of proportional or threshold-based systems.

The system achieves **natural control behavior**: stimulation activates only when voluntary effort is detected, ramps smoothly, and deactivates when the user relaxes. Healthy subjects showed that the controller behaves consistently and safely, while patient trials confirmed that it can reduce muscular effort without sacrificing accuracy in trajectory tracking. Together, the two phases establish a proof of concept that hybrid EMG-based control can close the loop between user intent and electrical stimulation in a functional way.

However, several **limitations** remain.
The first is **hardware-related**: although the algorithm does not need explicit M-wave computation, it still requires EMG recordings where the M-wave is visible. This demands high-quality amplifiers, separate stimulation and sensing electrodes, and careful electrode placement. If the acquisition system blanks or saturates during stimulation, the filter cannot estimate the correlated component, and performance degrades.
The second limitation is **calibration**. Each subject requires manual tuning of thresholds $(e_{ON}$, $e_{OFF}$ and scaling parameters to match their specific EMG amplitude range. This process is time-consuming and reduces usability in daily-life applications.
A third limitation lies in the **dependence on stable electrode positioning**. Slight shifts between sessions alter the recorded EMG amplitude and spectral content, forcing recalibration and reducing robustness over long-term use.
Finally, the experiments were mostly conducted under constrained laboratory conditions with an exoskeleton; how the system performs during unconstrained or dynamic daily activities remains to be validated.

The **next steps** that naturally follow this work are:
to design **automatic calibration and adaptive thresholding** procedures to make the controller self-tuning;
to integrate **artifact-tolerant hardware** capable of capturing M-waves even during shared electrode use;
and to extend testing to **daily-life scenarios** where subjects move freely, so that the system’s robustness and comfort can be assessed outside controlled settings.
More broadly, combining adaptive filtering with **machine learning–based intent estimation** could further enhance reliability, allowing future NMES systems to learn each user’s neuromuscular characteristics automatically and provide assistance that feels both natural and effortless.
