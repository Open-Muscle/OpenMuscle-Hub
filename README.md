# OpenMuscle Hub

Welcome to the central repository for all OpenMuscle devices, software, and documentation.  
OpenMuscle is a modular, open-source prosthetic sensing platform using pressure and muscle activity to drive machine learning–based control.

This repository serves as the **starting point** for users, contributors, and researchers exploring the OpenMuscle ecosystem.

---

## 🚀 Project Overview

OpenMuscle enables wearable biosensing devices using custom sensors, open firmware, and open hardware. The system is designed for researchers, developers, and prosthetics hackers interested in:

- **Real-time muscle signal acquisition**
- **Machine learning model training (classification/regression)**
- **Experimental prosthetic and robotic control**
- **3D-printable enclosures and modular design**

---

## 📂 Active Repositories

| Project | Description | Repository |
|--------|-------------|------------|
| **FlexGrid** | 60-sensor wearable with flexible PCB and pressure matrix | [OpenMuscle-FlexGrid](https://github.com/Open-Muscle/OpenMuscle-FlexGrid) |
| **LASK5** | Handheld labeler wand for training ML models on gesture inputs | [OpenMuscle-LASK5](https://github.com/Open-Muscle/OpenMuscle-LASK5) |
| **Band (OM-12)** | Original OpenMuscle armband with 12 sensors (legacy design) | [OpenMuscle-Band](https://github.com/Open-Muscle/OpenMuscle-Band) |
| **Software/Firmware** | Shared firmware for all devices, includes async scanning, model hooks, and examples | [OpenMuscle-Software](https://github.com/Open-Muscle/OpenMuscle-Software) |

---

## 📜 Documentation

📚 Full build guides, videos, and tutorials will be hosted here or at [https://openmuscle.org](https://openmuscle.org).  
This repository will grow into a full documentation hub with instructions for:

- Building each device
- Flashing and configuring firmware
- Training ML models with labeled data
- Mechanical assembly (with 3D-printable STL files)
- Example use cases and integration ideas

📌 **Check each device repo for its specific README and wiring guide.**

---

## 🛠️ Getting Started

🧪 Want to build a device or contribute?

1. **Choose a device** (e.g. FlexGrid or LASK5) and visit its repo.
2. Follow the hardware build guide in the `/Docs` folder.
3. Flash the firmware from [OpenMuscle-Software](https://github.com/Open-Muscle/OpenMuscle-Software).
4. [Optional] Integrate the data into your own ML workflow.

---

## 🧩 System Architecture

🧠 OpenMuscle uses a modular architecture:

- Devices like FlexGrid and LASK5 communicate using MicroPython firmware.
- Sensors send signals over ESP-NOW or Wi-Fi protocols.
- A Python or serial-based pipeline can log, visualize, or process signals.
- Data can be used in supervised ML pipelines for gesture classification or finger prediction.

> 🔧 A full system diagram will be added here soon.
>  
> 📷 Placeholder for an image/diagram:
> 
```markdown
![System Architecture Diagram](./assets/system-architecture-placeholder.png)
```

---

## 🌱 Roadmap (WIP)

| Milestone | Description | ETA |
|----------|-------------|-----|
| FlexGrid v1.0 | Finalize PCB, BOM, and test data quality | ⬜ Q4 2025 |
| LASK5 v5.0 | Usability testing and power optimization | ⬜ Q4 2025 |
| Documentation Hub | Flesh out full guides and build flow | ⬜ Q4 2025 |
| Public ML Datasets | Upload labeled data sessions | ⬜ TBD |

---

## 🤝 Contributing

We welcome pull requests, design files, firmware improvements, and documentation edits.

Start by reading the contribution guide:  
```markdown
📄 [CONTRIBUTING.md](./CONTRIBUTING.md)
```

You can also join upcoming GitHub Discussions to suggest ideas or improvements.

> ✅ Beginners welcome! Look for "good first issue" tags in the Software repo.

---

## 💬 Community (Coming Soon)

- GitHub Discussions
- Discord Server (invite link TBD)
- Monthly Community Builds & AMAs

> Want to feature your build or fork? Submit to the **Show & Tell** section in Discussions!

---

## 📄 License

All documentation and text in this repo is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/).  
Any example code snippets are licensed under the [MIT License](https://opensource.org/licenses/MIT).

```markdown
📄 See [LICENSE.md](./LICENSE.md) for full details.
```

---

## 🧭 About

OpenMuscle is developed by [@TURFPTAx](https://github.com/turfptax) and the broader open-source community.  
This project is part of an ongoing effort to make prosthetics more affordable, transparent, and hackable.

```markdown
🌐 More at [openmuscle.org](https://openmuscle.org)
```

