# SatQuery AI

**An Interactive Vision-Language Assistant for Multimodal Remote Sensing Image Analysis through Text Queries**

Problem Statement ID: **26167**
Organization: Indian Space Research Organisation (ISRO), Department of Space
Category: Software · Theme: Space Technology

---

## Table of Contents
1. [Overview](#1-overview)
2. [The Problem We Are Solving](#2-the-problem-we-are-solving)
3. [Our Solution](#3-our-solution)
4. [System Architecture](#4-system-architecture)
5. [Technical Approach](#5-technical-approach)
6. [Foundations We Build On](#6-foundations-we-build-on)
7. [Development Roadmap](#7-development-roadmap)
8. [Feasibility](#8-feasibility)
9. [Impact](#9-impact)
10. [Repository Structure](#10-repository-structure)
11. [Setup & Usage](#11-setup--usage)
12. [Team](#12-team)

---

## 1. Overview

SatQuery AI is an agentic vision-language assistant that lets a user upload one or two remote-sensing images and ask a question in plain English, then returns an evidence-grounded answer: visual proof (bounding boxes, change maps), a confidence estimate, and a transparent record of exactly which model or tool produced the answer. It removes the need for the user to understand satellite-data characteristics, GIS workflows, or model selection - the system figures that out and shows its work.

## 2. The Problem We Are Solving

Remote-sensing imagery underpins agricultural monitoring, disaster management, urban planning, forest and water-resource assessment, and infrastructure mapping. In practice, most existing remote-sensing AI tools are built as single-purpose applications - one model for land-cover classification, another for change detection, another for VQA - each requiring a user who already understands GIS workflows and sensor characteristics.

A further limitation is that many operational questions cannot be answered from one image alone. Optical/multispectral imagery gives spectral and contextual detail; synthetic aperture radar (SAR) gives complementary structural information and works through cloud cover and at night. Change over time requires paired multitemporal observations. A general-purpose vision-language model, without adaptation to remote-sensing data and terminology, cannot reliably handle any of this.

SatQuery AI addresses this by combining domain-adapted remote-sensing models with an **agentic controller** that interprets the query, validates the inputs, selects the correct specialist workflow, and integrates the results - rather than relying on one generic model to do everything.

## 3. Our Solution

SatQuery AI accepts three input configurations, each corresponding to a mandatory capability of the system:

| Input | Capability | Example Query |
|---|---|---|
| Single optical or SAR image | Visual question answering, plus captioning or text-guided region grounding | *"Describe the land cover and major objects visible in this image."* / *"Highlight the water body referred to in the query."* |
| Co-registered optical + SAR pair | Cross-modal fusion - extracting complementary information from both sensors | *"Use the optical and SAR images together to identify built-up and water-covered regions."* |
| Bi-temporal image pair | Change description and change-based visual question answering | *"What changed between these two dates, and where did the change occur?"* / *"Has the built-up area increased, decreased, or remained unchanged?"* |

Every response includes an **auditable execution summary**: the task the controller identified, the model(s)/tool(s) it invoked, the parameters used, and the confidence of the result. This is a deliberate design choice, not an afterthought - a system that cannot show its work is not one a district administration or disaster-response team can act on with confidence.

## 4. System Architecture

```
                     ┌─────────────────────────────┐
                     │        Web / GUI Layer        │
                     │  upload images + type query   │
                     └───────────────┬───────────────┘
                                      │
                     ┌───────────────▼───────────────┐
                     │   Input Validator / Preprocessor │
                     │  - format check (GeoTIFF/TIFF)   │
                     │  - modality detect (optical/SAR) │
                     │  - co-registration check         │
                     │  - metadata extraction           │
                     └───────────────┬───────────────┘
                                      │
                     ┌───────────────▼───────────────┐
                     │      Agentic Controller (LLM)    │
                     │  - classify task from query       │
                     │  - select model(s) from registry   │
                     │  - configure permitted parameters   │
                     │  - sequence execution                │
                     │  - produce auditable execution trace  │
                     └──────┬───────┬────────┬────────┘
              ┌─────────────┘       │        └─────────────┐
    ┌─────────▼────────┐  ┌────────▼────────┐  ┌───────────▼──────────┐
    │  Single-Image     │  │  Change/Bi-Temporal│  │  Optical–SAR Fusion │
    │  Specialist        │  │  Specialist         │  │  Specialist          │
    │  VQA + grounding    │  │  Change-VQA /        │  │  Cross-modal          │
    │  or captioning       │  │  change description   │  │  information extraction│
    └─────────┬────────┘  └────────┬────────┘  └───────────┬──────────┘
              └─────────────┬───────┴────────┬─────────────┘
                     ┌───────▼────────────────▼───────┐
                     │   Output Integrator              │
                     │  - merge text + visual evidence   │
                     │  - confidence estimation           │
                     │  - render boxes / change map         │
                     └───────────────┬───────────────┘
                                      │
                     ┌───────────────▼───────────────┐
                     │  Response: text + visuals +     │
                     │  execution summary + report      │
                     └─────────────────────────────────┘
```

The controller performs three roles that together make the system agentic rather than a single monolithic model: it interprets the query and classifies the requested task, it checks that the supplied inputs are actually compatible with that task before running anything, and it selects and sequences the specialist model(s) required - logging each decision into the execution trace it returns alongside the answer.

## 5. Technical Approach

**Domain adaptation.** A generic vision-language model does not understand remote-sensing terminology, sensor characteristics, or the visual vocabulary of satellite imagery. Our vision-language components are fine-tuned on BigEarthNet (co-registered Sentinel-1 SAR and Sentinel-2 multispectral imagery with text annotations), giving the system genuine remote-sensing grounding rather than a general-purpose model applied out of the box.

**Shared backbone with task-specific adapters.** Rather than training three unrelated models, we adapt a single vision-language backbone using low-rank adaptation (LoRA), with a dedicated adapter per specialist task. This keeps the specialists consistent in how they represent and reason about imagery while still allowing the controller to make a genuine, logged choice of which adapter and workflow to invoke for a given query - which is what the execution trace records.

**Cross-modal fusion.** Optical and SAR imagery are fundamentally different signal types - SAR is unaffected by cloud cover and darkness but represents structure and backscatter rather than color and texture. Our fusion strategy processes each modality through its appropriate specialist and integrates the two outputs into a single, evidence-grounded answer, directly reflecting how a human analyst would cross-reference the two image types.

**Change understanding.** Bi-temporal reasoning is trained against a well-defined answer space - change/no-change, increase/decrease, magnitude and category of change - allowing the system to give precise, verifiable answers to change queries rather than vague free-text descriptions, while still supporting descriptive change summaries where useful.

**Input validation.** Before any model is invoked, the controller verifies image count, modality, format (GeoTIFF/TIFF for operational use; PNG/JPEG only for the benchmark evaluation sets), and co-registration compatibility with the requested task, and reports a clear reason if the request cannot be fulfilled - rather than producing an unreliable answer from mismatched inputs.

## 6. Foundations We Build On

Our approach builds on and adapts established, published remote-sensing vision-language research rather than developing modeling techniques from first principles, which allows engineering effort to concentrate on the agentic orchestration layer - the part of the system unique to this problem statement.

| Foundation | Contribution to SatQuery AI |
|---|---|
| **BigEarthNet** | Primary dataset for remote-sensing domain adaptation across optical and SAR imagery. |
| **VRSBench** | Evaluation and fine-tuning data for single-image captioning, grounding, and VQA. |
| **RSVQA** | Evaluation benchmark for single-image visual question answering. |
| **CDVQA** | Evaluation and training data for multitemporal, change-based visual question answering, with a well-defined closed-answer space. |
| **GeoChat** (Grounded Large Vision-Language Model for Remote Sensing) | Reference architecture and fine-tuning methodology for grounded, remote-sensing-adapted single-image understanding. |
| **EarthDial** | Reference for staged multi-sensor, multi-temporal adaptation - demonstrating that a single vision-language system can be progressively trained to handle RGB, SAR, multispectral, and temporal imagery. |

## 7. Development Roadmap

| Phase | Focus | Outcome |
|---|---|---|
| **Phase 1** | Single-image baseline - VQA, plus captioning or grounding | Domain-adapted single-image specialist |
| **Phase 2** | Multitemporal change understanding | Change-VQA / change-description specialist |
| **Phase 3** | Optical–SAR cross-modal analysis | Fusion specialist producing integrated, evidence-grounded output |
| **Phase 4** | Agentic controller and interactive application | Full query-to-response pipeline with routing, validation, and execution trace |
| **Phase 5** | Evaluation and integration | End-to-end validation against public benchmark test splits |
| **Phase 6** | Finalization | Complete demonstration system, documentation, and deliverables |

## 8. Feasibility

The system is built to be deliverable within a realistic development timeline:

- Domain adaptation uses **LoRA fine-tuning of existing open checkpoints**, not training a foundation model from scratch - an approach already demonstrated at practical compute budgets by GeoChat and EarthDial.
- The training and evaluation datasets required for every mandatory task (BigEarthNet, VRSBench, CDVQA) are already public and prepared for exactly these tasks, removing data collection as a project risk.
- Cross-modal fusion, the most technically novel component, uses a **late-fusion strategy** - each modality is processed by its appropriate specialist and the outputs are integrated - avoiding dependence on a jointly-trained cross-modal network while still satisfying the requirement to combine complementary optical and SAR information.
- The agentic controller and input-validation layer are deterministic software components independent of model training progress, allowing the orchestration, GUI, and reporting layers to be developed and demonstrated in parallel with model adaptation.

## 9. Impact

- **Lowers the technical barrier to satellite data.** State agriculture departments, disaster-management authorities, and urban-planning bodies frequently have access to Cartosat and RISAT imagery but not to GIS-trained personnel. A natural-language interface makes that data directly usable.
- **Addresses a concrete operational gap in Indian conditions.** Monsoon cloud cover routinely blocks optical satellite observation for weeks at a time - precisely during the flood and cyclone season when timely imagery is most needed. RISAT SAR observes through cloud cover and at night; a system that reasons jointly over optical and SAR imagery closes this gap without requiring manual cross-referencing by a specialist analyst.
- **Generalizes across monitoring use cases.** The change-understanding capability applies equally to flood extent tracking, deforestation monitoring, urban encroachment, water-body change, and agricultural land-use shifts - a single reusable capability rather than a narrow, single-purpose tool.
- **Designed for institutional trust and adoption.** An auditable execution trace and visual evidence, rather than an opaque model output, is what allows a decision-maker to act on the system's answer with confidence - a necessary property for any AI tool intended for use in a government operational context.

## 10. Repository Structure

```
satquery-ai/
├── docs/                             # detailed technical planning notes
├── configs/                          # model and task configuration
├── data/
│   ├── raw/                          # datasets (not versioned)
│   └── sample/                       # sample inputs for demonstration
├── src/satquery/
│   ├── types.py                      # shared data contracts (task types, execution trace schema)
│   ├── controller/
│   │   ├── router.py                 # query-to-task classification
│   │   ├── validator.py              # input compatibility checking
│   │   └── agent.py                  # agentic orchestration and execution trace
│   ├── specialists/
│   │   ├── single_image.py           # VQA, captioning, grounding
│   │   ├── change_detection.py       # bi-temporal change understanding
│   │   └── fusion.py                 # optical–SAR joint analysis
│   ├── integration/                  # output and confidence integration
│   └── utils/image_io.py             # GeoTIFF/SAR/PNG input handling
├── app/main.py                       # interactive web application
├── scripts/                          # data preparation and pipeline verification
└── tests/                            # automated tests
```

## 11. Setup & Usage

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Verify the end-to-end pipeline
python scripts/sanity_check.py

# Prepare evaluation data
python scripts/inspect_data.py --dataset cdvqa

# Launch the application
python app/main.py

# Run automated tests
pytest tests/
```

---

*Developed for ISRO Problem Statement 26167 - SatQuery AI.*
