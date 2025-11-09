## Inspiration

The venture capital landscape is dynamic and fast-paced, yet the due diligence process often remains manual, time-consuming, and prone to human bias. We were inspired to create VentureAI to address this critical bottleneck. Our goal was to leverage the power of generative AI and agentic architectures to transform raw, unstructured founder materials (like pitch decks) and public data into actionable, structured investment insights. We envisioned a system that could democratize access to sophisticated analysis, allowing investors to make smarter, faster decisions, and founders to receive more insightful feedback.

## What it does

VentureAI is an AI-powered platform designed to automate and enhance the early-stage investment due diligence process. It acts as an intelligent assistant for venture capitalists, performing a suite of tasks:

* **Pitch Deck Analysis:** It ingests PDF pitch decks and intelligently extracts key information across various categories such as team, problem, solution, market opportunity, traction, business model, and financials.
* **Web Research & Verification:** Beyond the pitch deck, VentureAI conducts independent web research (e.g., using Google Search and LinkedIn scraping) to verify claims, enrich data, and gather additional context on market sizes, competitors, and founder backgrounds.
* **Structured Investment Memo Generation:** It synthesizes all gathered information into a comprehensive, structured investment memo, highlighting discrepancies between founder claims and external research, and providing a clear investment recommendation.
* **Investor Query Answering:** Investors can ask natural language questions about any specific analysis, and VentureAI will retrieve relevant data from its BigQuery data warehouse to provide informed answers.
* **Follow-up Question Generation:** To facilitate deeper due diligence, the platform can generate challenging and insightful follow-up questions for founders, designed to clarify ambiguities, address concerns, and validate key assumptions.

## Project Architecture

![Project Architecture Diagram](https://firebasestorage.googleapis.com/v0/b/valued-mediator-461216-k7.firebasestorage.app/o/VentureAI_cloud_run.png?alt=media&token=0f6236ba-471e-4bb5-86ee-6437353fc0ff)

## How we built it

VentureAI is built on a robust, serverless architecture primarily leveraging Google Cloud Platform (GCP) services and an agent-based design pattern.

### Agent Architecture (`manager_agent/agent.py`)

The core intelligence of VentureAI resides in its modular agent architecture, orchestrated by a central `root_agent` (also known as the Manager Agent). This design, built using the Google Agent Development Kit (ADK), allows for complex, multi-step tasks to be broken down and handled by specialized AI agents.

1. **`root_agent` (Manager Agent):**
    * This is the primary orchestrator. Its main role is to analyze the user's request and determine the appropriate sub-agent to handle the task.
    * It examines keywords and input types (e.g., presence of a PDF, specific questions) to identify the user's intent:
        * **Pitch Deck Analysis Task:** If a PDF is provided with keywords like "analyze" or "investment memo".
        * **Investor Question Task:** If a plain text question is asked.
        * **Follow-up Questions Task:** If keywords like "follow-up questions" are present.
    * It then delegates the task to one of its specialized sub-agents.

2. **`investment_memo_generation_agent` (Sequential Agent):**
    * This is a composite agent that executes a predefined sequence of sub-agents to generate a full investment memo.
    * **`pitch_deck_extractor_agent`:** This agent's sole purpose is to meticulously analyze the provided pitch deck content and extract all relevant claims and data points into a structured JSON format. It focuses on aspects like company name, team, problem, solution, market, traction, business model, competitive advantage, financials, and initial investment recommendations mentioned in the deck.
    * **`web_research_analyst_agent`:** This agent takes the extracted claims and uses the `google_search` tool to perform independent web research. It verifies claims, gathers additional public information (e.g., market sizes, competitor details, founder backgrounds), and enriches the data. It's instructed to cite sources with URLs.
    * **`report_generation_agent`:** Acting as a senior VC partner, this agent synthesizes the information from both the `pitch_deck_extractor_agent` and the `web_research_analyst_agent`. It highlights discrepancies between internal claims and external research, and generates the final, comprehensive investment memo. This agent uses Pydantic schemas to ensure its output is a strictly structured JSON object, guaranteeing data consistency.

3. **`investor_query_agent`:**
    * This agent is responsible for answering specific investor questions.
    * It utilizes a custom tool, `get_analysis_data`, to fetch relevant investment analysis data from Google BigQuery based on a provided `analysis_id` (retrieved from the session state).
    * It then formulates a clear and concise natural language answer to the investor's question using the retrieved data.

4. **`followup_questions_agent`:**
    * This agent generates challenging and insightful follow-up questions for founders.
    * Similar to the `investor_query_agent`, it uses the `get_analysis_data` tool to retrieve the full context of the investment analysis from BigQuery.
    * It then crafts a list of questions designed to clarify discrepancies, address concerns, and validate assumptions, outputting them in a structured JSON format with context and categories.

### Frontend using Firebase Studio

To provide an intuitive user experience, we have developed a dynamic website using Firebase Studio. This frontend seamlessly connects to the VentureAI agent backend, enabling users to interact with the system, upload pitch decks, view analysis results, and engage with the AI agents for queries and follow-up questions. The integration ensures a smooth and responsive user interface for all interactions.

### Docker Container and `docker_main.py`

The VentureAI application is containerized using Docker and deployed as a scalable, serverless service on Google Cloud Run. The `docker_main.py` file serves as the entry point for this containerized application, providing a robust API layer using FastAPI.

* **FastAPI Application:** `docker_main.py` initializes a FastAPI application, which exposes several HTTP endpoints for interacting with the VentureAI system. It also configures CORS to allow requests from various origins.
* **Agent Runner (`google.adk.runners.Runner`):** It instantiates a `Runner` with the `root_agent` and a `FirestoreSessionService`. The `Runner` is responsible for managing the execution of the agent, handling messages, and maintaining session state.
* **Session Management (`FirestoreSessionService`):** The `FirestoreSessionService` (defined in `manager_agent/firestore/firestore_session_service.py`) is crucial for persisting user session data, including conversation history and agent state, in Google Cloud Firestore. This ensures continuity across multiple interactions.
* **BigQuery and Google Cloud Storage Clients:** The `docker_main.py` sets up clients for Google BigQuery (for data warehousing of analysis results) and Google Cloud Storage (for storing generated PDF investment memos).
* **API Endpoints and Workflow:**
  * **`/create_session`:** Handles the creation or retrieval of user sessions, returning a unique `session_id` and the current session state.
  * **`/generate_investment_analysis`:** This is a central endpoint for initiating a pitch deck analysis.
        1. It receives a PDF URL, user ID, session ID, and other metadata.
        2. It downloads the PDF content.
        3. It constructs a message containing the PDF as an inline data part and a prompt for the agent.
        4. It calls `runner.run_async` to execute the `root_agent`. The `root_agent` then orchestrates the `investment_memo_generation_agent` (which includes extraction, web research, and report generation).
        5. Upon receiving the structured JSON output from the agent, it generates a PDF investment memo using `reportlab` and uploads it to a specified Google Cloud Storage bucket.
        6. The extracted and synthesized data is then inserted into Google BigQuery for persistent storage and analytics.
        7. Finally, it updates the session state in Firestore with details like the `analysis_id` and the URL of the generated PDF.
  * **`/get_investor_dashboard_data`:** Retrieves all stored investment analysis records from BigQuery, suitable for populating an investor dashboard.
  * **`/investor_query`:** Allows investors to ask questions about a specific analysis. It updates the session state with the `analysis_id` to focus the `investor_query_agent` and then sends the prompt to the `root_agent`.
  * **`/followup_question`:** Triggers the generation of follow-up questions by sending a specific prompt to the `root_agent`, which delegates to the `followup_questions_agent`.

### Deployment with `cloud_run.sh`

The `cloud_run.sh` script, developed by our team, streamlines the deployment of the VentureAI application to Google Cloud Run. This script automates the entire process, from building the application's Docker image to deploying it as a managed service.

The implementation involves several key steps:

1. **Docker Image Construction:** The script first builds a Docker image of the VentureAI application. This image encapsulates all the necessary code, dependencies, and configurations required to run the application in a consistent environment.
2. **Image Tagging and Registry Push:** Once the Docker image is built, it is tagged with a specific name and version, making it identifiable within Google Cloud's ecosystem. This tagged image is then pushed to Google Artifact Registry, a secure and private repository for storing Docker images.
3. **Cloud Run Service Deployment:** Finally, the script deploys the application to Google Cloud Run. It instructs Cloud Run to use the image stored in Artifact Registry. During this deployment, crucial environment variables are passed to the Cloud Run service. These variables, such as `GCS_BUCKET_NAME`, `PROJECT_ID`, `LOCATION`, `DATABASE`, and `GOOGLE_API_KEY`, are essential for the application to connect to other GCP services (like Google Cloud Storage, BigQuery, and Firestore) and function correctly. The deployment also configures the service to be publicly accessible, allowing external clients to interact with its API endpoints. The script also includes functionality to run the Docker container locally for development and testing purposes.

## Challenges we ran into

Building VentureAI presented several interesting challenges:

* **Orchestrating Complex Workflows with Agents:** Designing the `manager_agent` to effectively delegate tasks to multiple specialized sub-agents and manage the flow of information between them was crucial. Ensuring each agent performed its specific role without overlap or confusion required careful instruction prompting and state management.
* **Ensuring Structured Output from LLMs:** A significant hurdle was consistently obtaining clean, parseable JSON output from the generative models, especially for the `pitch_deck_extractor` and `report_generation_agent`. We addressed this by providing very explicit instructions, few-shot examples, and leveraging Pydantic schemas in the `report_generation_agent` to enforce strict output formats.
* **Designing a Comprehensive BigQuery Schema:** Storing the rich, nested data generated by the AI analysis in BigQuery required a flexible yet structured schema that could accommodate all relevant fields and be easily queried for various investor insights.
* **Real-time Web Research Integration:** Integrating external web search capabilities (Google Search) and a LinkedIn scraper into the agent workflow, while maintaining performance and relevance, was complex. This involved careful prompt engineering to guide the `web_research_analyst_agent` to extract specific, verifiable information.
* **State Management Across Interactions:** For multi-turn conversations (like investor queries or follow-up questions), maintaining session state and context across different API calls and agent interactions was critical. Firebase Firestore played a key role here.
* **Deployment and Environment Management:** Managing the deployment of the Dockerized FastAPI application to Cloud Run, along with its respective dependencies and environment variables, added a layer of operational complexity.

## Accomplishments that we're proud of

We are particularly proud of:

* **Exclusive Use of Google Technologies:** This project was built entirely using Google's robust technology stack, from leveraging the Agent Development Kit (ADK) for agent construction, utilizing the powerful Gemini model for AI capabilities, to deploying the entire application on Google Cloud Run. For session management, we developed a custom Firestore session service, and for storing comprehensive investor data, we rely on BigQuery. This end-to-end Google ecosystem approach ensures seamless integration, scalability, and performance.
* **End-to-End Automation:** Successfully building a fully automated pipeline from raw PDF pitch deck to a structured investment memo and actionable insights.
* **Robust Multi-Agent System:** The modular and scalable multi-agent architecture allows for easy expansion and improvement of individual specialized agents without disrupting the entire system, making the project highly adaptable and maintainable.
* **Actionable Insights Generation:** The ability to not only summarize but also to verify claims, highlight discrepancies, answer specific investor questions, and generate insightful follow-up questions truly adds value to the due diligence process.
* **Structured Data Output:** Achieving consistent and reliable structured JSON output from the generative models, which is crucial for downstream analysis and integration.

## What we learned

Throughout the development of VentureAI, we gained valuable insights into:

* **The Power of Agentic AI:** How breaking down complex problems into smaller, specialized agent tasks can lead to more robust and manageable AI applications.
* **Prompt Engineering for Structured Data:** The art and science of crafting effective prompts to guide large language models (LLMs) to produce specific, structured outputs.
* **Google Cloud AI Services:** Deepened our understanding and practical experience with BigQuery for analytical data warehousing, and Firebase for backend services.
* **Serverless Architecture Best Practices:** Designing for scalability, cost-efficiency, and maintainability in a serverless environment using FastAPI within Docker and deployed on Google Cloud Run.
* **Importance of Data Schema:** The critical role of well-defined data schemas (like those enforced by Pydantic) in ensuring data quality and consistency when working with generative AI.
* **Error Handling and Resilience:** The necessity of robust error handling, retry mechanisms, and logging in distributed, AI-driven systems.

## What's next for VentureAI

The future of VentureAI is exciting, with several key areas for expansion:

* **Enhanced Data Sources:** Integrate with more diverse data sources beyond Google Search and LinkedIn, such as financial databases, news APIs, and industry reports, to provide even richer context and verification.
* **Advanced Financial Analysis:** Develop specialized agents for deeper financial modeling, projection validation, and risk assessment based on financial statements and other quantitative data.
* **Founder Feedback Loop:** Create a mechanism for founders to receive automated, constructive feedback on their pitch decks based on VentureAI's analysis, helping them refine their narratives.
* **Sentiment Analysis and Trend Spotting:** Incorporate sentiment analysis on market commentary and news, and develop capabilities to identify emerging industry trends relevant to investment opportunities.
* **Multi-language Support:** Extend the platform to support pitch decks and research in multiple languages.
* **Integration with CRM/Deal Flow Systems:** Allow seamless integration with existing venture capital CRM or deal flow management platforms.
