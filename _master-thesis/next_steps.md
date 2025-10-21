---
title: "Next steps"
permalink: /master-thesis/next-steps
# excerpt: "How to quickly install and setup Minimal Mistakes for use with GitHub Pages."
# last_modified_at: 2021-06-07T08:48:05-04:00
redirect_from:
  - /theme-setup/
toc: true
---

## Understand if M-wave acquisition is feasible with our current equipment

In several works such as Ambrosini (2014) and Osuagwu (2020), the M-wave is explicitly used to model and remove stimulation artifacts from the EMG signal. These methods rely on the system’s ability to physically capture the M-wave, which represents the muscle’s direct electrical response to stimulation. Detecting it is essential for accurately separating physiological activity from artifact contamination.

However, not all sEMG systems are capable of recording the M-wave. Many commercial amplifiers are optimized for voluntary EMG rather than electrically evoked responses. They often include input blanking or protection circuits that temporarily disable the input stage during and immediately after the stimulation pulse to prevent damage from high-voltage transients. As a result, these systems may be unable to record both the M-wave and the post-stimulation voltage decay, since both occur within this protected interval.

Attempting to test M-wave acquisition on hardware not designed for stimulation scenarios risks overvoltage damage to the amplifier inputs. Therefore, before proceeding, it is necessary to confirm whether the current sEMG hardware supports evoked EMG acquisition (i.e., recording during or immediately after stimulation) or includes artifact rejection and fast recovery circuitry.
