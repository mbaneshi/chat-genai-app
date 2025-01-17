Okay, let's break down this project into manageable steps and outline the code structure. We'll build a FastAPI backend, a SQLAlchemy model, and a SvelteKit frontend to interact with the Google Gemini API.

**Project Structure:**

```
gemini_app/
├── backend/           (FastAPI backend)
│   ├── app/
│   │   ├── __init__.py
│   │   ├── database.py   (SQLAlchemy setup)
│   │   ├── models.py     (SQLAlchemy models)
│   │   ├── schemas.py    (Pydantic schemas)
│   │   ├── main.py      (FastAPI app)
│   │   ├── gemini_service.py   (Google Gemini API integration)
│   ├── alembic.ini       (Alembic config)
│   ├── migrations/      (Alembic migration scripts)
│   ├── requirements.txt
│
├── frontend/          (SvelteKit frontend)
│   ├── src/
│   │   ├── lib/
│   │   │   ├── api.js    (Frontend API interaction)
│   │   ├── routes/
│   │   │   └── +page.svelte
│   │   ├── app.html
│   │   ├── global.css
│   ├── package.json
│   ├── svelte.config.js
│   ├── vite.config.js
│
├── .env               (Environment variables)
└── README.md
```

**1. Backend (FastAPI with SQLAlchemy):**

**1.1 Install Dependencies:**

```bash
cd backend
pip install fastapi uvicorn sqlalchemy alembic pydantic python-dotenv google-generativeai
```

**1.2 `.env` file:**

```env
DATABASE_URL="sqlite:///./gemini.db"  # SQLite for simplicity, can be changed
GOOGLE_API_KEY="YOUR_GOOGLE_API_KEY" # Get from Google AI Studio
```

**1.3 `backend/app/database.py`:**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
import os
from dotenv import load_dotenv

load_dotenv()

DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_engine(DATABASE_URL)
Base = declarative_base()
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**1.4 `backend/app/models.py`:**

```python
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.sql import func
from .database import Base


class PromptResponse(Base):
    __tablename__ = "prompt_responses"

    id = Column(Integer, primary_key=True, index=True)
    prompt = Column(String, index=True)
    response = Column(String)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

**1.5 `backend/app/schemas.py`:**

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class PromptResponseBase(BaseModel):
    prompt: str

class PromptResponseCreate(PromptResponseBase):
    pass

class PromptResponse(PromptResponseBase):
    id: int
    response: str
    created_at: datetime

    class Config:
        orm_mode = True
```

**1.6 `backend/app/gemini_service.py`:**

```python
import google.generativeai as genai
import os
from dotenv import load_dotenv

load_dotenv()

genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))
model = genai.GenerativeModel("gemini-pro")


def generate_response(prompt: str) -> str:
    """Generates a text response using Google Gemini Pro model."""
    try:
        response = model.generate_content(prompt)
        return response.text
    except Exception as e:
        return f"Error generating response: {e}"
```

**1.7 `backend/app/main.py`:**

```python
from typing import List

from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session

from . import models, schemas
from .database import engine, get_db
from .gemini_service import generate_response
from fastapi.middleware.cors import CORSMiddleware

models.Base.metadata.create_all(bind=engine)


app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins for simplicity; configure in prod
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.post("/prompt", response_model=schemas.PromptResponse, status_code=status.HTTP_201_CREATED)
def create_prompt(prompt: schemas.PromptResponseCreate, db: Session = Depends(get_db)):
    response_text = generate_response(prompt.prompt)
    db_prompt_response = models.PromptResponse(prompt=prompt.prompt, response=response_text)
    db.add(db_prompt_response)
    db.commit()
    db.refresh(db_prompt_response)
    return db_prompt_response


@app.get("/prompts", response_model=List[schemas.PromptResponse])
def get_prompts(db: Session = Depends(get_db)):
    prompts = db.query(models.PromptResponse).all()
    return prompts
```

**1.8 `backend/requirements.txt`:**

```
fastapi
uvicorn[standard]
sqlalchemy
alembic
pydantic
python-dotenv
google-generativeai
```

**1.9 Alembic setup (optional for DB migration management):**

```bash
cd backend
alembic init migrations
```

Update `backend/alembic.ini` with your database URL. Then create initial migration:

```bash
alembic revision --autogenerate -m "create_prompt_response_table"
alembic upgrade head
```

**2. Frontend (SvelteKit):**

**2.1 Install Dependencies:**

```bash
cd frontend
npm create svelte@latest .  # Create a new SvelteKit project. Choose skeleton project
npm install axios
```

**2.2 `frontend/src/lib/api.js`:**

```javascript
import axios from "axios";

const API_URL = "http://localhost:8000"; // Replace if your API runs elsewhere

export const sendPrompt = async (prompt) => {
  try {
    const response = await axios.post(`${API_URL}/prompt`, { prompt });
    return response.data;
  } catch (error) {
    console.error("Error sending prompt:", error);
    throw error;
  }
};

export const fetchPrompts = async () => {
  try {
    const response = await axios.get(`${API_URL}/prompts`);
    return response.data;
  } catch (error) {
    console.error("Error fetching prompts:", error);
    throw error;
  }
};
```

**2.3 `frontend/src/routes/+page.svelte`:**

```svelte
<script>
    import { sendPrompt, fetchPrompts } from '$lib/api';

    let prompt = '';
    let response = null;
    let allPrompts = [];
    let errorMessage = null;

    const handlePrompt = async () => {
        try {
            response = await sendPrompt(prompt);
            errorMessage = null;
            prompt = "";
             await fetchAllPrompts();
        } catch (error) {
           response = null;
           errorMessage = "Failed to send prompt. Please try again.";
        }
    };


    const fetchAllPrompts = async () => {
        try {
           allPrompts = await fetchPrompts();
        } catch (error){
            errorMessage ="Failed to fetch prompts."
        }

    }

    fetchAllPrompts()
</script>

<div class="container">
    <h1>Gemini App</h1>
    <div class="form-container">
        <textarea bind:value={prompt} placeholder="Enter your prompt..." rows="5"></textarea>
        <button on:click={handlePrompt}>Send Prompt</button>
    </div>
    {#if errorMessage}
        <div class="error">{errorMessage}</div>
    {/if}

    {#if response}
        <div class="response-container">
            <h2>Response:</h2>
            <p>{response.response}</p>
        </div>
    {/if}

    <h2>Previous Prompts:</h2>
    <div class="prompt-list">
        {#each allPrompts as prevPrompt}
            <div class="prompt-item">
                <p><strong>Prompt:</strong> {prevPrompt.prompt}</p>
                <p><strong>Response:</strong> {prevPrompt.response}</p>
                <p class="small"> <i>Sent at: {new Date(prevPrompt.created_at).toLocaleString()}</i></p>
            </div>
        {:else}
          <p>No previous prompts yet.</p>
        {/each}
    </div>
</div>

<style>
   .container {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 20px;
		max-width: 800px;
		margin: 0 auto;
	}

	.form-container {
		display: flex;
		flex-direction: column;
		margin-bottom: 20px;
        width: 100%;
	}

    .form-container textarea {
      margin-bottom: 10px;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 16px;
      width: 100%;
      resize: vertical;
    }


	.form-container button {
		padding: 10px 15px;
		background-color: #007bff;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
        font-size: 16px;
        width: fit-content;
	}
    .form-container button:hover {
        background-color: #0056b3;
    }

	.response-container {
		background-color: #f0f0f0;
		padding: 15px;
		border-radius: 5px;
		margin-bottom: 20px;
        width: 100%;
	}


  .prompt-list {
        width: 100%;
        margin-top: 20px;
    }

    .prompt-item {
		border: 1px solid #ddd;
		padding: 15px;
		margin-bottom: 10px;
        border-radius: 5px;
    }


    .prompt-item p {
        margin-bottom: 5px;
    }

    .prompt-item .small {
        font-size: 0.8em;
         color: gray
    }


    .error {
        color: red;
        margin-bottom: 10px;
        font-size: 1.1em;
     }

     h1 {
      margin-bottom: 20px;
     }
</style>
```

**2.4 `frontend/package.json`:**

```json
{
  "name": "frontend",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "preview": "vite preview",
    "check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
    "check:watch": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json --watch"
  },
  "devDependencies": {
    "@sveltejs/adapter-auto": "^2.0.0",
    "@sveltejs/kit": "^1.27.4",
    "svelte": "^4.0.5",
    "svelte-check": "^3.4.3",
    "tslib": "^2.4.1",
    "typescript": "^5.0.0",
    "vite": "^4.4.2"
  },
  "dependencies": {
    "axios": "^1.6.2"
  }
}
```

**To run this application:**

1. **Set API key:** Add your google api key to `.env` file
2. **Backend:**
   ```bash
   cd backend
   pip install -r requirements.txt
   uvicorn app.main:app --reload
   ```
3. **Frontend:**
   ```bash
   cd frontend
    npm install
   npm run dev -- --open
   ```

**Explanation:**

- **Backend (FastAPI):**
  - Sets up database using SQLAlchemy.
  - Creates a `/prompt` endpoint to receive the user's prompt and return the response from Google Gemini API after storing it to the database.
  - `/prompts` endpoint to retrive previous prompt and response.
  - Uses `google-generativeai` library to interact with Google Gemini API.
- **Frontend (SvelteKit):**
  - Uses `axios` to make HTTP requests to the FastAPI backend.
  - Displays a form to take the prompt.
  - Displays previous prompts and current response.
  - Fetches all the saved prompts and responses from the backend.

**Next Steps:**

- **Error Handling:** Enhance error handling on both backend and frontend.
- **Loading state:** Display loading state on sending prompt and receiving response.
- **Styling:** Add more styling to improve look and feel.
- **Authentication/Authorization:** Add authentication if needed.
- **Deployment:** Containerize the app and deploy on cloud.

This should provide a good starting point for your project. Remember to replace placeholders with your actual API key, and modify the code as needed.
