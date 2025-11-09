# VentureAI: AI-Powered Investment Due Diligence Platform

## Project Overview

VentureAI is an AI-powered platform designed to automate and enhance the early-stage investment due diligence process. It transforms raw, unstructured founder materials (like pitch decks) and public data into actionable, structured investment insights. The goal is to democratize access to sophisticated analysis, allowing investors to make smarter, faster decisions, and founders to receive more insightful feedback.

## Features

VentureAI acts as an intelligent assistant for venture capitalists, performing a suite of tasks:

* **Pitch Deck Analysis:** Ingests PDF pitch decks and intelligently extracts key information across various categories such as team, problem, solution, market opportunity, traction, business model, and financials.
* **Web Research & Verification:** Conducts independent web research (e.g., using Google Search and LinkedIn scraping) to verify claims, enrich data, and gather additional context on market sizes, competitors, and founder backgrounds.
* **Structured Investment Memo Generation:** Synthesizes all gathered information into a comprehensive, structured investment memo, highlighting discrepancies between founder claims and external research, and providing a clear investment recommendation.
* **Investor Query Answering:** Investors can ask natural language questions about any specific analysis, and VentureAI will retrieve relevant data from its BigQuery data warehouse to provide informed answers.
* **Follow-up Question Generation:** Generates challenging and insightful follow-up questions for founders, designed to clarify ambiguities, address concerns, and validate key assumptions.

## Project Architecture

VentureAI is built on a robust, serverless architecture primarily leveraging Google Cloud Platform (GCP) services and an agent-based design pattern.

![Project Architecture Diagram](https://firebasestorage.googleapis.com/v0/b/valued-mediator-461216-k7.firebasestorage.app/o/VentureAI_cloud_run.png?alt=media&token=0f6236ba-471e-4bb5-86ee-6437353fc0ff)

### Agent Architecture

The core intelligence of VentureAI resides in its modular agent architecture, orchestrated by a central `root_agent` (Manager Agent) built using the Google Agent Development Kit (ADK). This design allows for complex, multi-step tasks to be broken down and handled by specialized AI agents:

* **`root_agent` (Manager Agent):** Orchestrates the workflow, analyzing user requests and delegating tasks to appropriate sub-agents (e.g., Pitch Deck Analysis, Investor Questions, Follow-up Questions).
* **`investment_memo_generation_agent`:** A composite agent that executes a sequence of sub-agents:
  * **`pitch_deck_extractor_agent`:** Extracts structured data from PDF pitch decks.
  * **`web_research_analyst_agent`:** Performs independent web research to verify claims and enrich data.
  * **`report_generation_agent`:** Synthesizes information and generates the final, comprehensive investment memo.
* **`investor_query_agent`:** Answers specific investor questions by fetching relevant analysis data from Google BigQuery.
* **`followup_questions_agent`:** Generates insightful follow-up questions for founders based on the investment analysis context.

### Backend (FastAPI & Docker)

The application is containerized using Docker and deployed as a scalable, serverless service on Google Cloud Run. The `docker_main.py` file serves as the entry point, providing a robust API layer using FastAPI.

* **FastAPI Application:** Exposes HTTP endpoints for interacting with the VentureAI system.
* **Agent Runner:** Manages the execution of the `root_agent` and handles messages and session state.
* **Session Management:** `FirestoreSessionService` persists user session data in Google Cloud Firestore, ensuring continuity across interactions.
* **Data Warehousing:** Utilizes Google BigQuery for storing analysis results and Google Cloud Storage for generated PDF investment memos.

## Technologies Used

* **Google Cloud Platform (GCP):** Cloud Run, Firestore, BigQuery, Cloud Storage
* **Google Agent Development Kit (ADK)**
* **Gemini Model**
* **FastAPI**
* **Docker**
* **Python**
* **Pydantic**
* **reportlab** (for PDF generation)

## Deployment

The `cloud_run.sh` script automates the deployment of the VentureAI application to Google Cloud Run.

### Prerequisites

1. **Google Cloud SDK:** Ensure you have the Google Cloud SDK installed and configured.
2. **Docker:** Docker must be installed on your machine.
3. **GCP Project:** A Google Cloud Project with billing enabled.
4. **Artifact Registry:** An Artifact Registry repository configured in your GCP project.
5. **Service Account:** A service account with necessary permissions (Cloud Run Admin, Storage Admin, BigQuery Admin, Firestore Admin) and its credentials configured for your local environment (e.g., `gcloud auth application-default login`).

### Local Execution

To build the Docker image and run the application locally:

```bash
./cloud_run.sh
```

This will start the FastAPI application on `http://localhost:8080`.

### Deploy to Google Cloud Run

To deploy the application to Google Cloud Run:

1. **Update Configuration:** Open `cloud_run.sh` and update the following variables with your GCP project details:
    * `GCP_PROJECT_ID`
    * `GCP_REGION`
    * `ARTIFACT_REGISTRY_REPO`
    * `CLOUD_RUN_SERVICE_NAME`
2. **Set Environment Variables:** Ensure the `set-env-vars` in `deploy_to_cloud_run` function within `cloud_run.sh` are correctly configured for your project, especially `GOOGLE_API_KEY`.
3. **Execute Deployment Script:**

    ```bash
    ./cloud_run.sh -r
    ```

    This script will:
    * Build the Docker image.
    * Tag the image for Google Artifact Registry.
    * Push the image to your specified Artifact Registry repository.
    * Deploy the image to Google Cloud Run as a managed service, making it publicly accessible.

## Challenges Faced

* **Orchestrating Complex Workflows with Agents:** Designing the `manager_agent` to effectively delegate tasks and manage information flow between specialized sub-agents.
* **Ensuring Structured Output from LLMs:** Consistently obtaining clean, parseable JSON output from generative models, addressed with explicit instructions, few-shot examples, and Pydantic schemas.
* **Designing a Comprehensive BigQuery Schema:** Creating a flexible yet structured schema for rich, nested AI analysis data.
* **Real-time Web Research Integration:** Integrating external web search and LinkedIn scraping while maintaining performance and relevance.
* **State Management Across Interactions:** Maintaining session state and context across different API calls and agent interactions using Firebase Firestore.
* **Deployment and Environment Management:** Managing Dockerized FastAPI application deployment to Cloud Run with dependencies and environment variables.

## Accomplishments

* **Exclusive Use of Google Technologies:** Built entirely using Google's robust technology stack (ADK, Gemini, Cloud Run, Firestore, BigQuery).
* **End-to-End Automation:** Successfully automated the pipeline from raw PDF pitch deck to structured investment memo and actionable insights.
* **Robust Multi-Agent System:** Modular and scalable multi-agent architecture allowing for easy expansion.
* **Actionable Insights Generation:** Ability to verify claims, highlight discrepancies, answer investor questions, and generate follow-up questions.
* **Structured Data Output:** Achieved consistent and reliable structured JSON output from generative models.

## What's Next for VentureAI

* **Enhanced Data Sources:** Integrate with more diverse data sources (financial databases, news APIs, industry reports).
* **Advanced Financial Analysis:** Develop specialized agents for deeper financial modeling and risk assessment.
* **Founder Feedback Loop:** Create a mechanism for founders to receive automated, constructive feedback.
* **Sentiment Analysis and Trend Spotting:** Incorporate sentiment analysis and identify emerging industry trends.
* **Multi-language Support:** Extend platform to support multiple languages.
* **Integration with CRM/Deal Flow Systems:** Allow seamless integration with existing venture capital CRM platforms.
