---
description: >-
  Overview of cmpunlocker for restoring CMP 170HX compute throughput and memory
  geometry.
---

# Current Unlock

The current community unlock is [cmpunlocker](https://github.com/amoghmunikote/cmpunlocker). It restores restricted SM compute throughput and HBM2e memory geometry on NVIDIA CMP 170HX (GA100) cards.

{% hint style="warning" %}
This project is highly experimental. Expect issues!
{% endhint %}

### Overview

The CMP 170HX uses a physically complete GA100 die. Its firmware and OTP configuration restrict compute and memory. cmpunlocker applies an in-driver unlock path each time the patched modules boot GSP for the CMP 170HX.

The tool unlocks the following features:

* 64GB (8GB) / 40GB (10GB) VRAM&#x20;
* Unthrottled compute performance
* PCIe Gen 2 speeds

Take a look at [cmpunlocker](https://github.com/amoghmunikote/cmpunlocker) for more information.

