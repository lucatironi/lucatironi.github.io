---
layout: page
title: Soul Nebula
---

<p class="page-meta">
  Westerhout 5 - <a href="/astro/">back to list</a>
</p>
[![Soul Nebula](/assets/astro/soul_nebula_mid.jpg)](/assets/astro/soul_nebula_big.jpg){:target="_blank"}
<p class="caption">
  click image to see full size (3008x3008px)
</p>
<p class="lead" markdown="1">
  The Soul Nebula, also known as Westerhout 5, is an emission nebula located in Cassiopeia, 7500 light years away from us. [Wikipedia](https://en.wikipedia.org/wiki/Westerhout_5)
</p>

## Acquisition Details

|-------|---|
| **Dates** | October 25th, 2022 (New Moon) |
| **Frames** | 50x300" (5mins), 4 hours and 10 minutes of total integration time |
| **Focal Length** | 287mm (f/4.7) |
| **Camera** | ZWO ASI533MC-Pro (gain 100, cooled at -10°C) |
| **Telescope** | TS-Optics Photoline 60mm Doublet Apochromatic Refractor |
| **Accessories** | 0.8x Flattener/Reducer |
| **Filters** | Optolong L-Extreme 2" Narrowband Filter |
| **Mount** | Sky-Watcher EQ6R-Pro |
| **Guide Scope** | Angeleyes Mini Guidescope 30mm f/4 |
| **Guide Camera** | ZWO ASI 120MM mini |
| **Computer** | ZWO ASIAIR Pro |

## Processing Details

I stacked the lights frames in [Siril](https://siril.org/) using the [OSC Preprocessing Without DBF (Darks, Biases, Flats)](https://free-astro.org/index.php?title=Siril:scripts) script. I decided not to use any Dark, Bias or Flat frames. I might try to create a dark frame library later, but the ASI533MC-Pro is considered a camera that can do without darks given the absence of amp glow.

I tried to process the image to replicate the so called "Hubble Palette" where Hydrogen gas usually has a orange hue and Oxygen gas is blue. At the bottom of the page there is an example of the same image but processed in a more "natural" way, where the nebula is appearing only red.

I then used the following tools:
- Photometric Color Calibration
- Remove Green Noise
- Generalized Hyperbolic Stretch (GHS) Transformations
- StarNet Star removal
- Split Channels
- RGB Compositing (Luminance->Red, Red->Red, Green->Red, Blue->Green)
- Color Saturation
- Star Recomposition with some stretching and reduction of the starmask

## Additional Images

<div id="gallery">
  <a class="gallery-item" href="/assets/astro/soul_nebula_natural_big.jpg" target="_blank">
    <img src="/assets/astro/soul_nebula_natural_small.jpg" alt="Soul Nebula Natural Colors">
    <div class="overlay">
      <div class="text">Soul Nebula with "natural" colors</div>
    </div>
  </a>
</div>