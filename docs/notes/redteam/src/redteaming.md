# Red Teaming

This page contains various notes on the `practice` of red teaming.

## TTPs

`TTPs` stands for tactics, techniques and procedures. This enables the description of modus-operandi an adversary uses to achieve a specific objective or group of objectives.
`Tactics` are the most high-level concept, and represent a top-level goal or objective, of which multiple are needed to achieve the ultimate goal. This could be moving laterally in an environment, or gaining initial access to it.
`Techniques` are the tradecraft an adversary leverages to achieve a specific tactic. For instance, if a tactic was initial access, then a technique might be phishing.
`Procedures` describe the specific implementation methodology of a technique. In keeping with the current example, a procedure for phishing might be device code phishing.

## Attack Lifecycle

This graph from Google Mandiant shows how individual TTPs form the attack lifecycle:

![](https://www.gstatic.com/bricks/image/eedab7cf-6618-4699-aa52-9c67e4428f8d.png)

## Emulation vs Simulation

Adversary Emulation and Simulation exercises are core red team tasks. 

Emulation refers to the process of assessing the TTPs employed by a particular adversary or cluster of adversaries, to subsequently deploy these specific TTPs against an organization in order to determine its preparedness in dealing with the TTPs of this specific threat actor (or cluster of threat actors). The actions taken by the red team directly mirror those which would have been taken by the specific threat actor(s) in question. Emulation is always informed by threat intelligence meant to actually scope what adversaries might target an organization, and what their respective TTPs are.
Simulation on the other hand allows more latitude in action for the red team. They may thereby act as a "fictional" adversary not constrained by existing patterns in the TTPs employed to simulate the attack.
Often the former preceeds the latter.

---

