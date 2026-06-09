# OpenMuscle Roadmap

**Status:** Living document, updated as milestones complete or strategy shifts.
**Last updated:** 2026-06-07
**Owner:** Tory Moghadam (TURFPTAx)
**Scope:** What OpenMuscle is building, in what order, and why.

This roadmap is intentionally honest about three things: where we are today, what's coming soon (with rough dates), and what the long-term vision looks like (without overselling). If you want to contribute, the near-term section is where to start. If you want to understand why we exist, read the mission and the ten-year vision.

For the underlying repo and firmware structure that supports this roadmap, see [docs/architecture-and-repo-layout.md](docs/architecture-and-repo-layout.md).

---

## Mission

Build accessible, affordable, open-source sensor technology that lets a wearable forearm device read what a human hand is doing, and put the designs, software, and data in the hands of anyone who wants to use them. The first beneficiaries are people with intact forearms but missing hands. The broader benefit is a shared, open dataset of human hand biomechanics that the entire research community can use.

OpenMuscle is not a product company. It is an open hardware project with two adjacent entities (a nonprofit and a separately disclosed for-profit) that together fund its growth without compromising the open core.

---

## Where we are today (June 2026)

- **FlexGrid V3 is working.** The 60-sensor (15 by 4) Velostat band runs end-to-end: live sensor capture, model training, inference, and integration with a robot hand. First clean inference session hit R² = 0.854.
- **FlexGrid V4 is in production.** Ten dev kits ordered, arriving mid-June 2026, shipping to two outside collaborators plus the founder.
- **LASK5** (the labeling wand for ground-truth capture) is functional and being migrated into the Open-Muscle GitHub organization.
- **Software stack** includes the PC application (Python, FastAPI web UI, ML training and hot-swap inference, robot hand forwarding) and the firmware in MicroPython on ESP32-S3.
- **First academic use:** a master's thesis at an Oregon university is testing FlexGrid on an amputee participant.
- **Legal structure:** not yet incorporated. Nonprofit and for-profit entities are planned (see Business Structure below).

---

## Near-term: next 6 to 12 months (mid-2026 through mid-2027)

The focus is on shipping V4 to outside hands, hardening the contribution flow so collaborators can start adding value, and standing up the public data pipeline.

### Hardware
- **FlexGrid V4 dev kit rollout.** Ten units ordered; first three (founder plus two collaborators) deployed by late June 2026. The V4 keeps the rigid PCB on the left side of the band, matching the V3 layout that proved out.
- **Bridge PCB research.** A separate small PCB that consolidates the current 19-wire harness down to 8 wires, intended for a future hardware revision (not V4 itself). Forward-looking R&D so the option exists when V5 design starts.
- **LASK5 hardware repo migration** from `turfptax/lask4` into `Open-Muscle/OpenMuscle-LASK5`, with hardware (CERN-OHL-S-2.0) and firmware (MIT, when promoted) cleanly split per the repo layout doc.

### Software and data
- **Multi-input training capture.** Add recording paths for keyboard input, game controller input, and musical instrument input (MIDI first). Every input becomes a labeled dataset where the bracelet sees the finger motion and the input device provides ground truth.
- **Public training data repository.** Stand up a hosted, downloadable corpus of labeled hand-biomechanics sessions, free to anyone for ML research. This is the long-game asset.
- **VR/XR client extraction.** Move the WebXR client out of the PC web app and into the new `OpenMuscle-AR` repo. PC app keeps the FastAPI server and the VR bridge endpoints.
- **CONTRIBUTING.md, Code of Conduct, and issue templates** finalized in the Hub repo so external contributors can open issues, route them to the right sub-repo, and get credit for ideas as well as code.

### Organization
- **Nonprofit incorporation** (501(c)(3) most likely). The nonprofit will own the OpenMuscle brand, the core open-source hardware designs, and the public datasets. Revenue comes from brand licensing fees, donations, and grants.
- **For-profit incorporation** (separately disclosed). A privately funded company that licenses the brand from the nonprofit and builds commercial products (initially a VR input device). Other companies are explicitly welcome to do the same. The nonprofit cannot, and will not, prevent commercial forks.

---

## Mid-term: 1 to 3 years (2027 through 2029)

The near-term work earns OpenMuscle the right to talk about scale. This section is what scale looks like.

### Hardware
- **FlexGrid V5** with the bridge PCB integrated, targeting a smaller, lighter, more wearable form factor. Goal is "all-day-wearable" comfort, not just "good for a session."
- **Per-device firmware repos** promoted as hardware stabilizes (FlexGridV4-Firmware, LASK5-Firmware, etc.), each pinned to a single hardware major.
- **First commercially-licensed third-party hardware.** Another company ships a product using OpenMuscle designs under the nonprofit's brand license. We help, they pay licensing fees, the nonprofit reinvests.

### Software
- **Mature PC application** with polished UI, packaged installer, multi-platform support.
- **Native VR headset application.** WebXR is the bridge; native Quest (or equivalent) APK is the target. Lives in `OpenMuscle-AR`.
- **Mobile / Android application** with Bluetooth sync to a paired bracelet. Phone becomes a portable training and capture device.
- **Inference at the edge.** Models small enough to run on the device or on the headset, not just on a tethered PC.

### Data and research
- **Public dataset at meaningful scale.** Target: tens of thousands of labeled sessions across diverse users, hand sizes, and tasks (typing, gaming, instruments, daily activities).
- **First peer-reviewed publications** beyond the Oregon thesis. Co-authored work with collaborating universities.
- **Open dataset citations** in third-party prosthetics, HCI, and BMI papers.

### Organization
- **Nonprofit board** filled with independent directors (prosthetics expertise, academic research, disability advocacy).
- **Self-sustaining revenue.** Licensing fees and donations cover operating costs without founder cash injection.
- **First nonprofit-funded device giveaways** to people who need a prosthetic-control sensor but can't afford one.

---

## Long-term vision: 5 to 10 years (2030 and beyond)

The further out a roadmap goes, the more it should describe outcomes rather than predictions. These are the outcomes we are building toward. They are conditional on a lot of things going right, and we say so openly.

### If OpenMuscle becomes the de facto standard for hand biomechanics sensing, the following becomes possible:

- **Large-scale consumer applications.** A hardware standard cheap enough and reliable enough to be used as a human interface device in games, productivity tools, creative software, accessibility tools, and XR experiences. Every use generates additional labeled training data, with explicit consent, that flows back into the public dataset.
- **Anti-cheat as a side effect.** Hand biomechanics are extremely hard to fake. A platform that reads what your hand is actually doing is, incidentally, a strong anti-cheat signal. Game publishers integrating this gain both an input modality and an integrity check. Players gain the option to contribute their gameplay biomechanics to prosthetic research.
- **Prosthetic AI training at unprecedented scale.** With orders of magnitude more diverse hand data than currently exists, machine learning models for prosthetic control improve dramatically. Models trained on this data improve quality of life for amputees, including veterans.
- **Brain-computer interface research enabled.** Companies and labs working on neural implants (Neuralink and the broader BMI field) need ground-truth movement data to correlate against neural signals. An open dataset of correlated hand biomechanics is one of the inputs that makes passive BCI viable. We will contribute data, not depend on partnership.
- **Free or subsidized prosthetic devices distributed globally** through the nonprofit, prioritizing regions where commercial prosthetics are unaffordable. The for-profit ecosystem subsidizes the giveaway program through licensing fees and donations.

### The Big Hairy Audacious Goal

Many people, around the world, using free open-source prosthetic hands made possible by OpenMuscle, regardless of their ability to pay.

This is the ten-year north star. Every milestone above is a step toward it. We may not get there in ten years. We are still building toward it.

---

## Business structure (disclosed)

OpenMuscle operates as two linked entities, by design:

| Entity | Purpose | Funding |
|---|---|---|
| **OpenMuscle Nonprofit** (planned 501(c)(3)) | Owns the brand, the open-source hardware designs (MIT), the public datasets, the GitHub organization | Donations, grants, brand-licensing fees |
| **OpenMuscle for-profit** (planned, name TBD, privately funded) | Licenses the brand from the nonprofit; builds commercial products; pursues regulated markets and contracts | Private investment |

Why both: a nonprofit alone cannot efficiently fund product development, regulatory work, or large-scale manufacturing. A for-profit alone risks closing the open core under shareholder pressure. The two-entity structure puts the core IP in the entity that structurally cannot sell it out, while letting the for-profit compete on execution.

**Other companies are explicitly welcome to build their own for-profit products on the open OpenMuscle designs.** The license is permissive (MIT for software, CERN-OHL-S-2.0 for hardware). The only thing the nonprofit gates is the trademark, which is what the for-profit brand license pays for.

---

## How this roadmap evolves

- **Quarterly review.** Once a quarter, this document gets updated to reflect what shipped, what slipped, and what changed.
- **Major decisions** (incorporation, repo restructuring, hardware revisions) are recorded as their own docs under `OpenMuscle-Hub/docs/` and linked from here.
- **Open discussion** happens in GitHub Discussions on this repo, and (when stood up) on the OpenMuscle Discord.
- **Contributors get credit.** Ideas surfaced via issues, even ones that don't ship, are tracked back to the contributor in commit co-author trailers, the CONTRIBUTORS file, or release notes.

If you have a use case, a critique, a proposed change to a milestone, or a question about why something is or isn't on this roadmap, [open an issue](https://github.com/Open-Muscle/OpenMuscle-Hub/issues/new). The Hub is the routing point.

---

## Glossary

- **FlexGrid:** the 60-sensor (15 by 4) Velostat-based pressure-sensing forearm band. Currently shipping V3 hardware, V4 in production.
- **LASK5:** the handheld labeling wand used to capture ground-truth finger positions while a user wears FlexGrid, for supervised ML training.
- **OpenMuscle-AR:** new repo for the WebXR client and future native VR/XR headset application.
- **PMG:** Pressure Myography. Reading muscle activity by sensing the pressure changes muscles cause under the skin.
- **TDMG:** Tissue Deformation Myography. Reading muscle activity by sensing how the muscle shape changes.
- **CERN-OHL-S-2.0:** Open hardware license used for OpenMuscle hardware. Permissive and reciprocal.
- **BHAG:** Big Hairy Audacious Goal. From Jim Collins's *Built to Last*. The ten-year north star.
