Title: Local AI API
Date: 2026-01-01
Category: GenAI
Tags: GenAI, FastAPI, llama.cpp, LocalAI, Python
Slug: local-ai-api
Status: Published

Today I connected FastAPI with a local LLM and built a simple AI API on my system. This was an important step because it joined backend development with local AI inference.

## Key Concepts

**FastAPI** — A Python framework used to build APIs quickly and clearly.

**Pydantic Schema** — A way to define input fields like prompt, max_tokens, and temperature.

**llama.cpp Integration** — Connecting a local GGUF model with FastAPI so the API can generate responses.

## What I Did

I loaded my GGUF model once and created a POST endpoint called `/generate`. This endpoint accepts a prompt and sends it to the local model. The response is returned as JSON.

I tested the API using Swagger UI and confirmed that the request and response flow worked correctly.

## Code Example

```python
from fastapi import FastAPI
from pydantic import BaseModel
from llama_cpp import Llama

app = FastAPI()

llm = Llama(
    model_path=r"C:\Users\Brindha Sivakumar\kact\7 Days GenAI Challenge\day5\llama\models\model.gguf",
    n_ctx=512,
    verbose=False
)

class PromptRequest(BaseModel):
    prompt: str
    max_tokens: int = 100
    temperature: float = 0.7

@app.post("/generate")
def generate_text(request: PromptRequest):
    response = llm(
        request.prompt,
        max_tokens=request.max_tokens,
        temperature=request.temperature
    )
    text = response["choices"][0]["text"].strip()
    return {"response": text}
```

## Output

```
{
  "response": "Artificial Intelligence is the science of making computers do things that humans can do."
}
```

## Learnings

- FastAPI can be connected with a local LLM  
- A model can be loaded once and reused for requests  
- Swagger UI makes API testing easy  
- Local AI APIs can work without cloud services  

## Challenges

- Faced event loop issues while running FastAPI in Jupyter  
- Understood that /generate works only with POST method  
- Improved prompt structure for better output  

## Conclusion

This task helped me understand how to turn a local model into a working API. It felt like building the base of a real AI product using my own system.

- Prompt goes in  
- Model generates response  
- API returns JSON output  

> Today I built a local AI API using FastAPI and llama.cpp.