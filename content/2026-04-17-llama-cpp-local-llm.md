Title: Running Local LLM using llama.cpp
Date: 2026-01-01
Category: GenAI
Tags: GenAI, LLM, llama.cpp, GGUF, LocalAI
Slug: running-local-llm
Status: Published

Today I explored something powerful — running a real Large Language Model directly on my laptop. No API keys, no cloud, no cost. This was a big shift from using online models to understanding how AI can work locally.

## Key Concepts

**llama.cpp** — A lightweight engine that allows running LLMs locally on your machine without needing internet or cloud services.

**GGUF Format** — A model file format optimized for fast loading and efficient inference, mainly used with local LLM tools.

**Quantization (Q4_K_M)** — A method to reduce model size while keeping performance usable, making it possible to run models on laptops.

## What I Did

I installed llama.cpp and verified the setup using the CLI. Then I downloaded a TinyLlama GGUF model and placed it in my local project folder.

Next, I ran the model using Python with the llama-cpp-python library. This allowed me to send prompts and receive responses directly inside JupyterLab.

## Code Example

```python
from llama_cpp import Llama

llm = Llama(
    model_path="C:/Users/Brindha Sivakumar/kact/7 Days GenAI Challenge/day5/llama/models/model.gguf",
    n_ctx=256,
    verbose=False
)

response = llm("What is Artificial Intelligence?", max_tokens=50)
print(response["choices"][0]["text"])
```
## Output

Artificial Intelligence is the development of systems that can perform tasks requiring human intelligence such as learning and decision making.

## Learnings

- Local LLM execution is possible without internet  
- GGUF models are optimized for local usage  
- Prompt design affects response quality  
- Small models need clear instructions  

## Challenges

- Faced file path errors while loading model  
- Initially got empty or incorrect responses  
- Understood importance of proper prompt structure  

## Conclusion

Running a local LLM helped me understand how AI systems actually work behind the scenes. It gave me confidence to build applications without depending on external APIs.

- My laptop can act as an AI engine  
- Local inference is fast and cost-free  

> Today I learned that powerful AI doesn’t always need the cloud — it can run right on your own machine.