Title: Local AI API with FastAPI
Date: 2026-04-18
Category: GenAI
Tags: GenAI, FastAPI, llama.cpp, LocalAI, Python
Slug: day6-local-ai-api
Status: Published

Today I built a local AI API by integrating FastAPI with a locally running LLM. This was an important step because it connected backend development with AI model execution.

Until now, I had used APIs and also tested a local model separately. But today, I combined both and created an endpoint where a prompt is sent and the model returns a response.

## What I Did

I created a POST endpoint called `/generate` using FastAPI. This endpoint accepts a prompt along with parameters like max_tokens and temperature.

The request is sent to a local GGUF model using llama.cpp, and the generated response is returned in JSON format.

The model is loaded once at startup, which improves performance and avoids reloading for every request.

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
- Model can be loaded once and reused  
- Swagger UI helps test APIs easily  
- Local AI APIs can work without internet  

## Challenges

- Faced event loop issues in Jupyter  
- Understood that /generate works only with POST method  
- Improved prompt clarity for better results  

## Conclusion

This task helped me understand how to turn a local AI model into a working API. It felt like building the foundation of a real AI system.

- Prompt goes in  
- Model generates response  
- API returns JSON output  

> Today I built a local AI API using FastAPI and llama.cpp.