# HR Policy Chatbot (RAG with .NET + Azure AI) — Project Plan

> Status: **planned**. Start after finishing the .NET fundamentals series and Azure basics (AZ-900 level).
> Goal: one hands-on, resume-worthy project that shows .NET backend, AI (RAG), Azure and CI/CD skills together.

---

## 1. The idea in one paragraph (simple version)

An employee asks: *"How many casual leaves do I get?"*
The app finds the right paragraph in the company's HR policy PDFs, gives that paragraph to an AI model, and the AI answers **only from it**, with a link to the source document.

That pattern is called **RAG: Retrieval-Augmented Generation**.

- **Retrieval** = search our own documents for the relevant pieces.
- **Augmented** = add those pieces to the question we send to the AI.
- **Generation** = the AI writes the answer.

**Analogy:** an open-book exam. The AI is the student, the HR policy is the book, and our search is the student flipping to the right page before answering.

Why not just ask ChatGPT? It doesn't know *our* company's policy, and it can make things up. RAG grounds the answer in our documents and shows where it came from.

We keep the running example from the OOP notes: **Employee, Manager, Developer**.
- An **Employee** asks about leave, WFH, reimbursement.
- A **Manager** can also see manager-only policies (appraisal, approvals).
- An **HR Admin** uploads and updates policy documents.

---

## 2. Scope

### MVP (must have)
1. HR Admin uploads policy documents (PDF / DOCX / TXT).
2. The app splits each document into chunks, creates embeddings, and stores them in Azure AI Search.
3. Employee asks a question in a chat UI; the API returns an answer **with citations** (document name + section).
4. If nothing relevant is found, the bot says *"I couldn't find this in the HR policies"* instead of guessing.
5. Chat history per user (last few turns kept so follow-up questions work).
6. Login with Microsoft Entra ID; role-based access (Employee vs Manager vs HRAdmin).
7. Everything deployed to Azure through a CI/CD pipeline.

### Nice to have (after MVP)
- Hybrid search (keyword + vector) with semantic ranking.
- Streaming answers (tokens appear as they are generated).
- Thumbs up/down feedback stored in SQL, shown on a small admin dashboard.
- Tool calling: "How many leaves do **I** have left?" calls a Leave API instead of the documents.
- Evaluation: a set of 20 to 30 test questions with expected answers, run in the pipeline.

### Out of scope
- Training or fine-tuning a model. (RAG means we don't need to.)
- A fancy frontend. A simple Blazor or React page is enough; the backend is the star.

---

## 3. Architecture

```
                 ┌──────────────────────────┐
  Employee ───▶  │  Chat UI (Blazor/React)  │
                 └────────────┬─────────────┘
                              │ HTTPS + Entra ID token
                              ▼
                 ┌──────────────────────────┐        ┌─────────────────────┐
                 │  ASP.NET Core Web API    │──────▶ │ Azure SQL Database  │
                 │  (App Service/Container) │        │ users, chats,       │
                 │                          │        │ feedback, documents │
                 │  /api/chat               │        └─────────────────────┘
                 │  /api/documents (admin)  │
                 └──┬───────────────────┬───┘
       1. search    │                   │ 2. question + found chunks
                    ▼                   ▼
        ┌────────────────────┐   ┌──────────────────────┐
        │  Azure AI Search   │   │  Azure OpenAI        │
        │  (vector index of  │   │  chat model +        │
        │   policy chunks)   │   │  embedding model     │
        └─────────▲──────────┘   └──────────────────────┘
                  │ chunks + embeddings
        ┌─────────┴──────────┐   ┌──────────────────────┐
        │ Ingestion worker   │◀──│ Azure Blob Storage   │
        │ (BackgroundService │   │ (uploaded PDFs)      │
        │  or Azure Function)│   └──────────────────────┘
        └────────────────────┘

  Secrets: Azure Key Vault + Managed Identity (no keys in code)
  Logs:    Application Insights
  Infra:   Bicep files     CI/CD: GitHub Actions
```

### The two flows

**A. Ingestion (HR Admin uploads a policy)**
1. File is saved to Blob Storage; a row is added in SQL (`Documents` table).
2. Worker extracts text, splits it into chunks (~500 to 800 tokens, small overlap).
3. Each chunk is sent to the embedding model, which returns a vector (a list of numbers that captures meaning).
4. Chunk text + vector + metadata (document name, section, allowed roles) are stored in Azure AI Search.

**B. Question answering (Employee asks)**
1. API turns the question into a vector with the same embedding model.
2. Azure AI Search returns the top 3 to 5 most similar chunks, filtered by the user's role.
3. API builds a prompt: system rules ("answer only from the context, cite sources, say you don't know otherwise") + chunks + chat history + question.
4. Azure OpenAI chat model writes the answer; API returns answer + citations.

### Solution structure (clean architecture, reuses the OOP + SOLID + DI lessons)

```
src/
  HrBot.Api/              Controllers / minimal APIs, auth, DI setup
  HrBot.Application/      Use cases: AskQuestion, IngestDocument; interfaces
  HrBot.Domain/           Entities: Employee, Manager, PolicyDocument, ChatMessage
  HrBot.Infrastructure/   EF Core (Azure SQL), Azure AI Search, Azure OpenAI, Blob clients
  HrBot.Worker/           Ingestion background service
tests/
  HrBot.UnitTests/        xUnit + Moq/NSubstitute
  HrBot.IntegrationTests/ WebApplicationFactory, Testcontainers for SQL
infra/                    Bicep templates
.github/workflows/        CI and CD pipelines
```

Key interfaces (where DI and the D in SOLID show up):
`IEmbeddingService`, `IVectorSearch`, `IChatCompletionService`, `IDocumentTextExtractor`, `IChunker`.
Swapping Azure AI Search for another store later = new class, no change to use cases.

---

## 4. Tech stack

| Area | Choice | Why it matters on the resume |
|---|---|---|
| Language / framework | C#, .NET 8 or newer, ASP.NET Core Web API | Core skill for the target roles |
| AI orchestration | **Microsoft.Extensions.AI** (or Semantic Kernel) | Microsoft's standard .NET AI abstractions |
| LLM + embeddings | **Azure OpenAI** (via Azure AI Foundry): a GPT chat model + `text-embedding-3-small` | Enterprise AI on Azure |
| Vector search | **Azure AI Search** (vector + hybrid search) | The "R" in RAG |
| Database | Azure SQL + **EF Core** | Ties in with the EF Core series |
| File storage | Azure Blob Storage | Common Azure service |
| Auth | Microsoft Entra ID, role-based authorization | Security, matches the cloud security goal |
| Secrets | Key Vault + Managed Identity | No secrets in code or pipeline |
| Hosting | Azure App Service (or Azure Container Apps with Docker) | Deployment skills |
| Infra as Code | **Bicep** | DevOps skill |
| CI/CD | **GitHub Actions** (Azure DevOps Pipelines as an alternative) | DevOps skill |
| Monitoring | Application Insights, structured logging | Production thinking |
| Testing | xUnit, Moq/NSubstitute, WebApplicationFactory | Quality |
| Frontend | Blazor (stay in C#) or a small React page | Keep it simple |

**Cost tip:** use the Azure free account credits, the **Free tier of Azure AI Search**, the cheapest App Service plan, and the serverless tier of Azure SQL. Delete or stop resources when not using them. Check current pricing before creating anything.

---

## 5. Milestones (step by step, one at a time)

Each milestone ends with something working, a commit, and a short note in this folder.

| # | Milestone | What you learn | Done when |
|---|---|---|---|
| 0 | **Prerequisites** | .NET fundamentals series, AZ-900 basics, Azure free account, Git basics | You can create a Web API and deploy a hello-world to App Service by hand |
| 1 | **Plain Web API + EF Core** | Clean architecture, EF Core, Azure SQL (local SQL first) | `POST /api/documents` saves document metadata; unit tests pass |
| 2 | **First AI call** | Azure OpenAI resource, Microsoft.Extensions.AI, prompts | `POST /api/chat` returns an AI answer (no RAG yet) |
| 3 | **Ingestion pipeline** | Blob Storage, text extraction, chunking, embeddings | Upload a PDF; chunks + vectors stored in Azure AI Search |
| 4 | **RAG answers with citations** | Vector search, prompt grounding, "I don't know" handling | Ask "How many casual leaves?" and get the right answer + source |
| 5 | **Auth and roles** | Entra ID, JWT, policies, role filters in search | Manager sees manager-only policies; Employee does not |
| 6 | **CI pipeline** | GitHub Actions: build, test, code coverage, lint | Every push/PR runs build + tests automatically |
| 7 | **Infra as Code + CD** | Bicep, environments (dev/prod), OIDC login from GitHub to Azure, Managed Identity, Key Vault | Merge to `main` deploys infra + app to Azure with no manual steps |
| 8 | **Observability** | Application Insights, logging, tracking token usage and latency | Dashboard shows requests, errors and AI cost per day |
| 9 | **Polish** | Chat UI, streaming, feedback, evaluation tests | Live demo link + README with architecture diagram + screenshots |

Rough time: 4 to 6 weeks at 1 to 2 hours a day, after the prerequisites.

### CI/CD in simple words
- **CI (Continuous Integration):** every time you push code, a robot builds it and runs the tests. If something breaks, you know in minutes.
- **CD (Continuous Deployment):** when code is merged to `main` and tests pass, the robot deploys it to Azure.

Planned pipeline:
```
push / PR ──▶ restore ──▶ build ──▶ unit tests ──▶ integration tests ──▶ publish artifact
merge to main ──▶ (CI above) ──▶ Bicep deploy (infra) ──▶ deploy app to dev ──▶ smoke test ──▶ manual approval ──▶ prod
```
Security touches (useful for the cloud security path later): OIDC federated login instead of stored Azure passwords, Key Vault for secrets, dependency scanning (Dependabot), secret scanning.

---

## 6. Resume bullet points (fill in real numbers when built)

**Project: HR Policy Assistant — AI chatbot using RAG | .NET 8, Azure OpenAI, Azure AI Search, GitHub Actions**

- Built an AI-powered HR policy assistant using **ASP.NET Core Web API** and **Retrieval-Augmented Generation (RAG)**, answering employee questions from company policy documents with source citations.
- Designed a document ingestion pipeline (Blob Storage → chunking → **Azure OpenAI embeddings** → **Azure AI Search** vector index) processing __ documents / __ pages.
- Implemented hybrid (keyword + vector) search with role-based filtering, so Employees and Managers only retrieve policies they are allowed to see; secured the API with **Microsoft Entra ID**.
- Applied **Clean Architecture, SOLID and dependency injection** with **EF Core** on Azure SQL; reached __% unit test coverage with xUnit.
- Automated build, test and deployment with **GitHub Actions** and **Bicep** (Infrastructure as Code), using OIDC and **Managed Identity + Key Vault** so no secrets are stored in code.
- Added monitoring with **Application Insights** to track latency, errors and AI token cost; reduced average response time from __ s to __ s.

Interview talking points this project gives you: what RAG is and why, chunk size trade-offs, embeddings and vector similarity, preventing hallucinations, prompt injection risks, cost control, how the pipeline deploys safely.

---

## 7. Before we start, check these off

- [ ] Finished the .NET fundamentals series
- [ ] Azure basics (AZ-900 topics: resource groups, App Service, Storage, SQL, Entra ID)
- [ ] Azure free account created, budget alert set (e.g. ₹500)
- [ ] Azure OpenAI access available in your subscription
- [ ] New GitHub repo created for the project (e.g. `hr-policy-chatbot`)

When these are done, say "ready" in the project thread and we start Milestone 1, one step at a time.
