Here’s a well-structured **assignment for Day 1: Foundation and First Steps with FastAPI**, designed to reinforce key concepts through hands-on practice and reflection:

------------------------------------------------------------------------

# 📘 **Assignment: Day 1 – Foundation and First Steps with FastAPI**

## 🎯 **Objective**

By the end of this assignment, you should be able to:

- Set up a FastAPI project environment

- Create and run a basic FastAPI application

- Understand and explore FastAPI’s automatic documentation features

------------------------------------------------------------------------

## 📝 **Tasks**

### 1. **Environment Setup**

Create a new project folder named `fastapi_day1`

Set up a virtual environment and activate it

Install FastAPI and Uvicorn:

``` bash
pip install fastapi uvicorn
```

### 2. **Create Your First FastAPI App**

Create a file named `main.py`

Implement a basic FastAPI app with one route:

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}
```

### 3. **Run and Explore**

Run the app using:

``` bash
uvicorn main:app --reload
```

Open your browser and explore:

Swagger UI at `http://127.0.0.1:8000/docs`

ReDoc at `http://127.0.0.1:8000/redoc`

### 4. **Add Another Endpoint**

Add a new route `/hello/{name}` that returns a personalized greeting using a path parameter.

### 5. **Reflection Questions**

Write short answers (2–3 sentences each):

- What are the key benefits of using FastAPI over Flask or Django for APIs?

- How does FastAPI use Python type hints to improve development?

- What did you find most interesting or surprising about the auto-generated docs?

------------------------------------------------------------------------

## 📂 **Submission**

Submit the following:

- `main.py` file

- Screenshot of Swagger UI showing both endpoints

- Answers to the reflection questions in a `.txt` or `.md` file

------------------------------------------------------------------------

Would you like me to generate a downloadable version of this assignment (PDF or Markdown)?
