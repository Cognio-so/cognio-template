# CLAUDE.md

This document provides **project guidelines and best practices** for all team members contributing to agentic Python projects using FastAPI backend, LangGraph agentic architecture, and Next.js frontend. It serves as the definitive guide for Claude Code development and integrates specific requirements for LLM orchestration, prompt management, and containerized deployment. Based on the project requirements given in implement.md , build the project following the guidelines expressed here. 

## Project Overview

- **Project Type:** Dockerized monorepo for agentic systems using Python 3.12, FastAPI backend, LangGraph agent flows, and Next.js frontend
- **Repository Structure:** Designed as a template for rapid prototyping and scaling of LLM-based agentic products
- **Primary Goals:** Flexibility, maintainability, optimal LLM selection, and seamless containerized deployment
- **Architecture:** Microservices approach with Docker Compose orchestration
-**Actual Project Requirements and SCope:** Understnad from implement.md 

## Repository Structure (Monorepo)

```
project-root/
├── backend/                    # FastAPI backend
│   ├── agents/                # Agent definitions and orchestration
│   ├── prompts/               # Modular prompt templates (one per agent)
│   ├── langgraph_flows/       # LangGraph workflows and subflows
│   ├── api/                   # FastAPI endpoints and routers
│   ├── core/                  # Core utilities and configurations
│   ├── models/                # Pydantic models for request/response
│   ├── services/              # Business logic services
│   ├── tests/                 # Backend tests
│   ├── pyproject.toml         # Python dependencies
│   ├── .python-version        # Python version (3.12)
│   ├── Dockerfile             # Backend Docker configuration
│   └── main.py               # FastAPI application entry point
├── frontend/                   # Next.js frontend
│   ├── src/
│   │   ├── components/        # Reusable React components
│   │   ├── pages/            # Next.js pages
│   │   ├── hooks/            # Custom React hooks
│   │   ├── utils/            # Utility functions
│   │   └── types/            # TypeScript type definitions
│   ├── public/               # Static assets
│   ├── package.json          # Node.js dependencies
│   ├── next.config.js        # Next.js configuration
│   └── Dockerfile            # Frontend Docker configuration
├── guides/                    # Documentation and guides
│   └── sandpack.md           # Sandpack integration guide
├── docker-compose.yml        # Multi-container orchestration
├── .env.example              # Environment variables template
├── .gitignore               
├── README.md
└── CLAUDE.md                # This file
```

## Python Environment & Dependency Management

- **Python Version:** 3.12 (specified in `backend/.python-version`)
- **Dependency Management:** Single `backend/pyproject.toml` for all Python dependencies
- **Local Development:**
  ```bash
  cd backend
  uv install -e .
  ```
- **Production:** Dependencies managed through Docker containers

## Model Integration & LangChain Library Mapping

**CRITICAL:** Always use the correct LangChain integration for each LLM family:

| Model Family | LangChain Library |
|--------------|-------------------|
| GPT (4o, 4o-mini, o3, o3-mini) | langchain-openai |
| Gemini series | langchain-google-genai |
| Open Source Models (llama 3.3 70b, kimi k2, Qwen3 32b, gemma 2) | langchain-groq |
| Claude series | langchain-anthropic |

### Required Dependencies in pyproject.toml

```toml
[project]
name = "agentic-backend"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.104.0",
    "langgraph>=0.2.0",
    "langchain-core>=0.3.0",
    "langchain-openai>=0.2.0",
    "langchain-anthropic>=0.2.0",
    "langchain-google-genai>=0.1.0",
    "langchain-groq>=0.1.0",
    "uvicorn[standard]>=0.24.0",
    "pydantic>=2.5.0",
    "python-dotenv>=1.0.0",
]
```

## Prompt Template Best Practices

### Modular Prompt Organization
- **Location:** Each agent's prompt lives in `backend/prompts/`
- **Naming Convention:** `{agent_name}_prompt.py`
- **Implementation:** Use `ChatPromptTemplate` from `langchain_core.prompts`

#### Example Structure:
```python
# backend/prompts/ui_agent_prompt.py
from langchain_core.prompts import ChatPromptTemplate

ui_agent_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful UI agent..."),
    ("human", "{user_input}"),
    ("assistant", "I'll help you create...")
])
```

### Prompt Management Benefits
- **Easy Editing:** Non-technical team members can modify prompts without touching agent logic
- **Version Control:** Track prompt changes through Git
- **Testing:** Isolated prompt testing and validation
- **Reusability:** Share prompts across different agents

## Agent Architecture & LangGraph Integration

### Core Principles
- Use appropriate LangChain library for each model
- Implement async patterns with `ainvoke` for all LLM calls
- Organize workflows in LangGraph for complex agent orchestration
- Keep agents modular and testable

## Asynchronous Best Practices

### Core Requirements
- **Always use `ainvoke`** for agent and LLM calls to ensure scalable, non-blocking operations
- Implement proper error handling for async operations
- Use concurrent processing where applicable

## Docker Configuration & Deployment

### Backend Dockerfile
```dockerfile
# backend/Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv for fast dependency management
RUN pip install uv

# Copy dependency files
COPY pyproject.toml .
COPY .python-version .

# Install dependencies
RUN uv install -e .

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend Dockerfile
```dockerfile
# frontend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Build the application
RUN npm run build

# Expose port
EXPOSE 3000

# Start the application
CMD ["npm", "start"]
```

### Docker Compose Configuration
```yaml
# docker-compose.yml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    volumes:
      - ./backend:/app
    depends_on:
      - db
    restart: unless-stopped

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000
    depends_on:
      - backend
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=agentic_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

volumes:
  postgres_data:
```

## Frontend Integration (Next.js + Sandpack)

### Core Requirements
- **TypeScript:** Use throughout the frontend for type safety
- **Modular Components:** Build reusable, stateless components
- **API Integration:** Clean separation between frontend and backend communication

## Environment Configuration

### Environment Variables Template
```bash
# .env.example

# Backend API Keys
OPENAI_API_KEY=your_openai_key
ANTHROPIC_API_KEY=your_anthropic_key
GOOGLE_API_KEY=your_google_key
GROQ_API_KEY=your_groq_key

# Application Configuration
ENVIRONMENT=development
DEBUG=true
DATABASE_URL=postgresql://postgres:postgres@db:5432/agentic_db

# Frontend Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_ENVIRONMENT=development
```

## Development Workflow

### Quick Start (Local Development)
1. **Clone and Setup:**
   ```bash
   git clone <your-template-repo>
   cd <project-name>
   cp .env.example .env
   # Fill in your API keys in .env
   ```

2. **Backend Setup:**
   ```bash
   cd backend
   uv install -e .
   ```

3. **Frontend Setup:**
   ```bash
   cd frontend
   npm install
   ```

4. **Run Development:**
   ```bash
   # Terminal 1 - Backend
   cd backend && uvicorn main:app --reload

   # Terminal 2 - Frontend  
   cd frontend && npm run dev
   ```

### Docker Development & Production
1. **Development with Docker:**
   ```bash
   docker-compose up --build
   ```

2. **Production Deployment:**
   ```bash
   docker-compose -f docker-compose.yml up -d
   ```

3. **Access Services:**
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:8000
   - API Documentation: http://localhost:8000/docs

### Adding New Agents
1. Create prompt file in `backend/prompts/new_agent_prompt.py`
2. Implement agent class in `backend/agents/new_agent.py`
3. Add to LangGraph flow in `backend/langgraph_flows/`
4. Create FastAPI endpoint in `backend/api/`
5. Add frontend integration in `frontend/src/`

## Testing & Quality Assurance

### Backend Testing
- Use `pytest` for unit and integration tests
- Test prompt templates and agent responses
- Mock external API calls for reliable testing

### Frontend Testing
- Use Jest and React Testing Library
- Test component rendering and user interactions
- Integration tests for API communication

## Code Style & Conventions

### Backend (Python)
- **Formatting:** Use `black` for code formatting
- **Type Annotations:** Use throughout for better code quality
- **Environment Variables:** Never hardcode secrets, always use `.env`

### Frontend (TypeScript)
- **TypeScript:** Use strict mode for better type safety
- **Components:** Functional components with hooks
- **State Management:** Use React context and hooks appropriately

## Best Practices Summary

### ✅ DO
- **Use modular prompts** in separate files for easy editing
- **Follow async patterns** with `ainvoke` for all LLM calls
- **Use correct LangChain libraries** for each model family
- **Containerize everything** with Docker for consistent deployments
- **Use TypeScript** throughout the frontend
- **Leverage Sandpack** for live code previews
- **Keep environment variables secure** and never commit them

### ❌ DON'T
- **Don't hardcode API keys** - always use environment variables
- **Don't call model APIs directly** - use the agent layer
- **Don't mix synchronous and asynchronous patterns**
- **Don't skip type annotations** in Python code
- **Don't commit sensitive data** to version control
- **Don't run production without containers**

## Docker Best Practices

### Development
- Use volume mounts for hot reloading during development
- Keep containers lightweight and focused
- Use multi-stage builds where appropriate

### Production
- Use specific image tags, not `latest`
- Implement health checks for all services
- Use secrets management for sensitive data
- Monitor container resources and logs

## Support & Collaboration

- **Code Reviews:** All prompt edits and agent logic changes require PR review
- **Documentation:** Update this CLAUDE.md file when adding new patterns
- **Docker:** Ensure all changes work in containerized environment
- **Testing:** Test both local and Docker environments before deployment

---

This template provides a solid foundation for building scalable, maintainable, and deployable agentic applications. The Docker Compose setup ensures consistent environments across development, staging, and production while maintaining the flexibility to develop locally when needed.