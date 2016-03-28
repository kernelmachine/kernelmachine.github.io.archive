---
layout: post
title: On Biologically Plausible Deep Learning
comments: true
---

Deep learning ultimately aims to mimic the function of the mammalian cortex. While extremely sought after, I'd like to argue that we are far, far out of reach of that goal.
Not only are most top performing deep learning algorithms supervised, while animal sensory learning is primarily unsupervised, or semi-supervised at best. In actuality, much of
our neural plasticity happens latently, when we are not conscious. Our brains are spontaneous and recurrent, hindering efforts to mimic it algorithmically. In this post,
I'll break down classical perspectives on feedforward architecture of the sensory experience, and present evidence that an entirely new plane of neural thought exists.

## Feedforward sensory experience

The thalamus is widely considered the neural gateway of sensory experience. Feedforward projections of the thalamus were originally proposed by Hubel and Wiesel in their
classic experiments with the cat visual cortex (Hubel and Wiesel, 1962).In their model, functional selectivity of cells in the primary visual cortex (V1), the area of initial visual processing in the neocortex, is determined by the alignment of receptive fields in the lateral geniculate nucleus (LGN), and subsequent feedforward projections into the visual area.  Their model is supported by studies that show highly specific, causal, and monosynaptic input of the LGN into V1 (Tanaka et al, 1983; Reid et al, 1995) and that inactivation of V1 leaves geniculate
 synaptic input functional (Ferster et al, 1996).  These studies suggest that cortical activity is primarily driven by external inputs.

## Recurrence and Spontaneous Activity


However, a lot of data also conflicts with this model. Suppression of GABA interneurons in the visual cortex degrades orientation tuning (Tsumoto et al, 1979; Sillito et al 1980), orientation-selective excitatory input to cortical cells mediates by feedback (Ahmed et al 1994), and tuning width is insensitive to the contrast of the stimulus
(Skottun et al, 1987). Anatomical evidence shows that less than 10 percent of connections within the cat visual cortex are from LGN, and that synapses onto spiny
stellate cells originate from layer VI pyramids and other spiny stellate cells (Douglas et al, 1995). A significant amount of recurrent, excitatory connectivity
exists within the neocortex (Douglas et. al, 1995). As a result, theories have emerged that propose functional selectivity originates from not only feedforward
thalamic input but also intrinsic cortical circuitry (Ben-Yishai et al, 1995).

Great insight into this debate has been brought about by investigations into spontaneous cortical activity (Steriade et al 1993; Luczak et al 2012). Spontaneous activity
is a core regime of neural operation (Kreiman et al 2000; Kosslyn et al 2001; Kraemer et al 2005; Sadaghiani et al 2010; Buzsaki, 1989). Because spontaneous circuit
activations are not initiated by external stimuli, they reflect dynamics mediated purely by intracortical circuitry. A prominent characteristic of spontaneous activity
is the common fluctuation in population spiking activity, called cortical states (Harris et al, 2011). In fact, early work on spontaneous activity delineated two major
cortical states that follow sleep cycle dynamics: synchronized or desynchronized states (Steriade et al 1993). Synchronized states are characterized by strong low-frequency
fluctuations in population cortical activity, and desynchronized states are characterized by a suppression of low-frequency fluctuations. Desynchronization is theorized
to provide the neural system with more flexibility in response (Harris et al, 2011). However, recent studies have shown that spontaneous activity forms a continuum of
states, and that intracortical activity patterns in the awake animal are not well described by oscillations due to their irregular spatiotemporal structure (Harris et al, 2011). Even so, patterns of spontaneous activity are most distinguishable during slow wave oscillations (SWOs), which are observable during slow wave sleep, quiet wakefulness, and under anesthesia (Luczak et al, 2012). SWOs are composed of prolonged (100 ms to several seconds) periods of spontaneous depolarization (UP states) and hyperpolarization (DOWN states) seen in intracellular recordings of cortical neurons (Metherate et al, 1992; Steriade et al, 1993). These single-cell states correspond to phases of highly active and quiescent population activity, respectively.

Definitive evidence of the importance of intrinsic cortical circuitry to emergent population activity came with the study of spontaneous circuit activations from
thalamic or sensory input (Tsodyks et al 1999; Kenet et al 2003; Fiser et al 2004; MacLean et al 2005; Eggermont 2006; Watson et al 2008; Luczak et al 2009;).
 Sensory or thalamic stimulation of sensory areas exhibit spontaneous activity that corresponds to UP states in multiple intracortical neurons and ensembles,
 and that spontaneous activity evoked by thalamic stimulation and by the intrinsic circuit are statistically indistinguishable. These studies suggest that
 intracortical circuitry is the dominant mediator of both spontaneous and evoked activation patterns. In other words, the primary role of thalamic input
 (and by extension external stimuli) might be to trigger different *pre-defined* circuits in sensory regions.

However, these results have not totally discredit the importance of thalamic input in evoking a sensory response. In fact, though thalamocortical connections
account for less than 10 percent of input in sensory regions, these connections are more reliable and produce greater post-synaptic depolarizations than
intracortical connections do (Gil et al 1999; Stratford et al, 1996). In addition, corticothalamic feedback is known to play a significant role in sensory
processing, highlighting the significance of thalamic excitation in intracortical activation patterns (Alitto et al, 2003). Thus there is probably a
combination of both thalamic drive and spontaneous intracortical dynamics that propagates an animal’s interaction with the external world.

The idea that meaningful neural activity only occurs when a stimulus is presented conflicts with the fact that the brain is highly active during sleep, development,
and other stages in which external stimuli is not detectable. All in all, findings that thalamically-evoked spontaneous activity is indistinguishable
from circuit-generated spontaneous activity suggest a couple things:

1) Even though in-vitro slice preparations of sensory regions are isolated from the rest of the brain, intracortical circuit activation patterns
  in these slices can provide insight into intracortical dynamics in-vivo. Understanding topological characteristics of the cortex is as important
  as understanding function.

2) The classic feedforward model of sensory architecture (e.g. Retina -> LGN -> V1) does not fully capture the system,
   since the cortex contains recurrent connections to the thalamus that can modulate sensory response, and corticocortical connections
   can modulate an evoked response in a spontaneous, state-dependent manner.
