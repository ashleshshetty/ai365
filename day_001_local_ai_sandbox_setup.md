# Day 001: Local AI Sandbox Setup, Leaderboards & Model Selection
**Date:** 2026-10-02 | **Category:** #Hardware #Quantization #Leaderboards #LMStudio | **Environment:** AWS Windows (NVIDIA Tesla T4 15GB VRAM) & Apple M1 Pro (32GB RAM)

## 1. Core Objective
* Establish a local AI testing sandbox on an AWS Tesla T4 GPU and M1 Pro MacBook to evaluate state-of-the-art open-weight models within a strict 15 GB VRAM hardware limit.

## 2. Key Concepts & Mental Models
* **Engine vs. Interface:** Local LLMs run on low-level backends like `llama.cpp`. Tools like [Ollama](https://ollama.com) serve as lightweight, background CLI plumbing for APIs and pipelines, while [LM Studio](https://lmstudio.ai) provides a full desktop GUI for visual model discovery, parameter tuning, and testing.
* **Quantization Trade-offs:** Raw models (`f16`) use 16-bit floating-point numbers per weight (~29 GB for a 14B model). Quantization compresses weights to 4-bit (`Q4_K_M`) using integer approximation, reducing the footprint to ~8.5 GB with minimal loss in reasoning quality.
* **Base vs. Instruct Models:** Base models are raw text approximators used for fine-tuning. Instruct/Chat models have undergone RLHF (Reinforcement Learning from Human Feedback) to operate as conversational assistants.
* **VRAM Context Overhead:** Model weights are only part of the VRAM footprint; active context tokens (KV cache) consume additional memory. Extreme context models (e.g., Qwen 1M) require hundreds of gigabytes of VRAM to hold 1M tokens in memory, making them unviable on single consumer/cloud GPUs.

## 3. Hands-On & Commands
* Verify NVIDIA CUDA driver status on Windows:
  ```cmd
  nvidia-smi
  ```
  *Output Verified:* Driver 553.24 | CUDA Version 12.4 | Tesla T4 (15360 MiB total VRAM) | Current Usage: 494 MiB.

* Check if Ollama is pre-installed on the system:
  ```cmd
  ollama --version
  where ollama
  ```

* Optimal LM Studio configuration for Tesla T4:
  1. Set Runtime Execution Engine to **CUDA 12** (`Ctrl+Shift+R`).
  2. Drag **GPU Offload** slider to **Max** to pin 100% of model layers onto VRAM.

## 4. Hardware Impact & Trade-offs
* **Tesla T4 (15 GB VRAM Limit):** Hard ceiling. A 14B model quantized to `Q4_K_M` (~8.57 GB) fits cleanly, leaving ~6.4 GB for KV cache and context overhead. A 27B or 32B model at `Q4_K_M` (~16–18 GB) exceeds VRAM and spills into system RAM, causing severe token generation slowdowns.
* **Power & Compute Scaling:** Idle draw is 17W / 70W at 12% utilization (Windows shell background processes). Full LLM inference pushes power toward 70W max TBP and utilization near 100%.

## 5. 30-Second Revision Flashcards
* **Sweet Spot Quantization:** `Q4_K_M` reduces VRAM footprint by >70% (~8.5 GB for 14B) while retaining ~98% of uncompressed intelligence.
* **15 GB VRAM Limit Rule:** Stick to 14B parameter models (`Q4_K_M`) for complete GPU acceleration without memory spilling.
* **Model Selection Rule:** Always download `-Instruct` or `-Chat` variants for interactive assistant workflows; ignore raw `-Base` models.
* **Leaderboard Strategy:** Use [LMSYS Arena](https://lmarena.ai) for subjective human quality rankings; use the [Hugging Face Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard) for objective benchmark filters (`Only Official Providers`, `Chat`, excluding `Merge`).

## 6. Exhaustive Deep-Dive & Detailed Notes
### Hardware Diagnostics
The system inspection via `nvidia-smi` confirmed driver version `553.24` with native CUDA `12.4` execution support. The Tesla T4 GPU operates on a PCI-E interface with 15,360 MiB (15.0 GB) GDDR6 memory capped at a 70W maximum power envelope. System idling used 494 MiB under standard WDDM display drivers (`explorer.exe`, `TextInputHost.exe`, `dcvagent.exe`).

### Leaderboard Navigation & Evaluation Strategy
1. **[LMSYS Chatbot Arena](https://lmarena.ai):** Evaluates models using crowdsourced blind A/B preference testing with Elo ratings. Features practical human-centric categories ("Best Coding Agents", "Best WebDev Models", "Hard Prompts").
2. **[Hugging Face Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard):** Runs automated scripts evaluating math (MATH, GPQA), reasoning (BBH, MUSR), and instruction-following (IFEval). 
   * *Why pages show "Archived":* Static evaluation datasets suffer from benchmark saturation (plateauing) as newer models achieve near 100% accuracy. Hugging Face periodically retires older benchmark suites and publishes new versions with steeper difficulty curves.
   * *Filtering out community noise:* Default Hugging Face views display thousands of user-submitted merges and experimental fine-tunes. To view flagship releases, apply:
     * Check **`Only Official Providers`** (top-right box).
     * Uncheck **`Merge`** under Model Type.
     * Select **`Chat`** under Model Type.

### Model Architecture Breakdown (Qwen Family Case Study)
When major open-source labs release model families, they publish structured tiers:
* **General Assistant:** `Qwen2.5-14B-Instruct` / `Qwen3-14B-Instruct` (all-round reasoning, text generation, 29+ languages).
* **Code Specialized:** `Qwen2.5-Coder-14B-Instruct` (trained on 5.5T code tokens; optimal for Cursor/IDE integrations).
* **Math Specialized:** `Qwen2.5-Math-7B/72B` (optimized for Chain-of-Thought and Tool-Integrated Reasoning).
* **Vision-Language (VL):** Multimodal engines supporting OCR, dense chart parsing, and long video inputs.

### GGUF & Quantization Mechanics
GGUF (GPT-Generated Unified Format) is a single-file binary format optimized by the `llama.cpp` project for rapid CPU/GPU offloading.
* `f16` (~29.55 GB): Full float precision. Exceeds 15 GB VRAM.
* `Q8_0` (~15.70 GB): 8-bit quantization. Exceeds 15 GB GPU ceiling.
* `Q5_K_M` (~10.51 GB): 5-bit quantization. Fits, but limits KV context headroom.
* `Q4_K_M` (~8.57 GB): 4-bit quantization using medium-complexity quantization matrix (`imatrix`). Optimal tradeoff for 15 GB GPUs.

### Web Search Integration
Local LLMs are isolated by default. LM Studio supports local plugins using free search endpoints (e.g., DuckDuckGo). The plugin performs background web queries, extracts HTML text, and prepends context to the prompt, keeping model processing 100% local while enabling real-time data access.

## 7. Tomorrow’s Next Step
* Load `Qwen2.5-Coder-14B-Instruct-GGUF` (`Q4_K_M`) into LM Studio, set up the local OpenAI-compatible API server on port `1234`, and connect it to a local coding editor extension (e.g., Continue.dev or Cursor).