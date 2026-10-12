---
title: "AI Model Local Deployment"
date: 2026-10-11
weight: 1
---

### LM Studio - Runs LLMs + GUI Chat interface
### VS Code/ Windsurf Integration - Continue Extension - open-source AI code agent
- find the confuguration of `continue extention in the file continue/.config 
```json
{
  "models": [
    {
      "title": "Local Qwen 14B",
      "provider": "lmstudio",
      "model": "qwen2.5-coder-14b-instruct",
      "apiBase": "http://192.168.122"
    }
  ]
}
```

### Specs: 1TB Disk | 64GB RAM | 12 Cores | 4GB Dedicated Graphics Card - Laptop1
- This machine can comfortably handle highly complex reasoning for advanced code modification and structured red teaming pipelines.

        | TASK                              | OPTIMIZED MODEL & QUANTIZATION     |
        |-----------------------------------|------------------------------------|
        | Weaponization & Complex Exploits  | Qwen2.5-Coder-32B-Instruct (Q4_K_M)|
        | Fast Payload Editing & Fuzzing    | Qwen2.5-Coder-14B-Instruct (Q8_0)  |
        | Zero-Refusal C2 & Command Exec    | Dolphin-2.9-Llama3-8B (Q8_0)       |

* **Why these configurations?**
  - Qwen2.5-Coder-32B (Q4_K_M): 
    - The model size is ~20GB. It runs brilliantly in your 64GB RAM. 
    - Set your engine to offload 4 to 5 layers to your 4GB GPU to accelerate the initial token processing speed (prompt ingestion). 
    - It has the reasoning power required to handle deeply obfuscated logic.
  - Dolphin-2.9-Llama3-8B (Q8_0): 
    - Because you have a dedicated GPU, you can run a higher quality quantization (Q8_0 instead of Q4) for the 8B model. 
    - The file will fit almost entirely inside your 4GB VRAM, resulting in blistering-fast command line output.


### Specs: 1TB Disk | 32GB RAM | 8 Cores | Integrated Graphics Only - Laptop2
- Because there is no dedicated GPU and your RAM is capped at 32GB, we prioritize high-speed efficiency and low-memory footprints so your system does not freeze during active operations.

    | TASK                              | OPTIMIZED MODEL & QUANTIZATION     |
    |-----------------------------------|------------------------------------|
    | Weaponization & Exploit Building  | Qwen2.5-Coder-14B-Instruct (Q4_K_M)|
    | Fast Script Obfuscation           | Qwen2.5-Coder-7B-Instruct (Q4_K_M) |
    | Zero-Refusal / Uncensored Chat    | Dolphin-2.9-Llama3-8B (Q4_K_M)     |

* **Why these configurations**?
  - Qwen2.5-Coder-14B (Q4_K_M): 
    - Takes up ~9GB of RAM. 
    - On pure CPU execution, this is the realistic ceiling for getting smart, multi-line code generation without waiting minutes for a response.
  - Dolphin-2.9-Llama3-8B (Q4_K_M): 
    - Lowering the quantization to Q4 drops the file size to ~4.8GB. 
    - This keeps the execution incredibly snappy on standard CPU threads, leaving roughly 25GB of free system RAM for your operating system and web browser.


