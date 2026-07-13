# Where to Buy a CMP 170HX

The CMP 170HX is no longer manufactured and is available exclusively on the secondhand market. Since Ethereum mining became unprofitable after The Merge in September 2022, these cards have flooded resale markets globally. This page covers current pricing, what to look for, where to buy, and what to avoid.

**Current Pricing (April 2025)**

Prices vary significantly by region and condition:

| Region             | Price Range       | Notes                                               |
| ------------------ | ----------------- | --------------------------------------------------- |
| USA (eBay)         | $200–400          | Wide variance; some listings still priced high      |
| Europe (eBay)      | €200–350          | Similar range to US                                 |
| China (secondhand) | $150–300          | Cheapest source; shipping costs offset savings      |
| Alibaba/bulk       | $400–500 per unit | Bulk only (10+ units); not worth it for individuals |
| UnixSurplus (US)   | \~$259            | US-based dealer with 90-day warranty                |

{% embed url="https://www.ebay.com/sch/i.html?_nkw=nvidia+cmp+170hx&_sop=12" %}

Pricing has been fairly stable in the $200–400 range since mid-2023. Cards have not continued dropping dramatically because the floor has largely been reached — below $200, sellers aren't motivated to ship them.

**What to Look For**

The CMP 170HX comes in two memory configurations that are occasionally confused in listings:

The **8GB HBM2e** version is the original and most common. This is the card documented throughout this GitBook — GA100-105F die, 3,584 CUDA cores (56 SMs), 4,096-bit memory bus, 1,493 GB/s bandwidth.

Some listings advertise a **10GB HBM2e** variant, which is a confirmed production variant with 4,480 CUDA cores (70 SMs), 5,120-bit memory bus, and approximately 1,865 GB/s bandwidth. Verify the actual VRAM and core count in your listing before purchasing.

When evaluating a listing, look for:

**Part number 900-11001-0105-000** — this is the official NVIDIA part number for the CMP 170HX 8GB. It appears on the PCB and in some listings.

**Passive cooling intact** — the card has no fans and relies on the aluminum/copper shroud for heat dissipation. Make sure the shroud is present and undamaged. Listings selling "board only" without the shroud will require aftermarket cooling immediately.

**Mining hours** — these cards were often run 24/7 in mining farms. High-hour cards may have degraded thermal paste and worn capacitors. There's no reliable way to check this from a listing, but cards from reputable dealers with return policies are safer.

**Avoid listings that:**

* Show PCB damage or burnt components
* Are missing the PCIe bracket (needed for proper watercooling installation)
* Claim gaming or display output capability (this is impossible on a real 170HX)
* Are priced above $400 unless they include warranty, accessories, or watercooling

**Where to Buy**

**eBay** is the most accessible source for buyers in the US and Europe. Search for "NVIDIA CMP 170HX" sorted by lowest price + shipping. Filter to "used" condition. Listings from established sellers with good feedback and return policies are preferable. Prices as of April 2025 range from $200 for bare boards to $350–400 for complete cards with accessories.

**UnixSurplus** (unixsurplus.com) is a US-based dealer in server hardware that stocks the CMP 170HX at approximately $259 with a 90-day seller warranty. This is a good option for US buyers who want some protection.

**AliExpress and Chinese Taobao** sellers offer the lowest prices but add shipping costs and longer delivery times. If you're buying multiple cards, Chinese sources can be significantly cheaper.

**What You'll Also Need**

The CMP 170HX is a passive server card designed for datacenter rack use. For workstation use you will need:

**A separate display GPU** — the CMP 170HX has no display outputs. You need another GPU or integrated graphics to drive a monitor. Any GPU will do, including a cheap GT 730 or the integrated graphics on your CPU.

**A PCIe x16 slot** — the card uses a standard PCIe connector and works in any x16 physical slot. It will operate electrically at x4 Gen 1 regardless of slot capabilities, so slot choice doesn't matter for performance currently.

**Adequate power supply** — the card draws up to 300W with the power limit raised. A quality 750W+ PSU is recommended if you're running it alongside other components.
