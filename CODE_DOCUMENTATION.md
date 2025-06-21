# Code-Level Documentation

## Overview
This document provides a detailed, code-level explanation of the FastAPI markdown analysis application. It is intended for developers who need to understand the internal workings of the service, from the API endpoint down to the individual functions and classes.

---

## 1. `main.py`
This file is the entry point of the FastAPI application. It defines the API server, middleware, and the primary endpoint for analyzing markdown files.

### Dependencies
- `fastapi`: The web framework.
- `pydantic`: For data validation and settings management.
- `models`: Contains Pydantic models for request/response bodies.
- `wst__markdown_processor`: Contains the `Wst_MarkdownExtractor` and `Wst_MarkdownHarmonizer` classes for processing markdown.
- `wst_product_config`: Configures and initializes the CrewAI agents.
- `shared_state`: A singleton for sharing data between different parts of the application.
- `utils`: A module containing helper functions for various tasks.
- `app_logging`: Configures application-wide logging.

### `app = FastAPI()`
Initializes the main FastAPI application instance.

### `app.add_middleware(CORSMiddleware, ...)`
Configures Cross-Origin Resource Sharing (CORS) middleware. This is necessary to allow web frontends from different domains to communicate with the API. It is currently configured to allow all origins, methods, and headers.

### `async def analyze_markdown(request: MarkdownAnalysisRequest)`
This is the core and only endpoint of the application.

- **Route**: `POST /analyze_markdown`
- **Request Body**: It expects a JSON object that conforms to the `MarkdownAnalysisRequest` model (defined in `models.py`), containing `markdown_text`, `product`, and `auth`.
- **Authentication**: Authentication is performed manually at the beginning of the function by calling `verify_auth_token(request.auth)`. The token is passed inside the request body.

#### Workflow:
1.  **Authentication**: The `auth` token from the request body is passed to the `verify_auth_token` function from `utils.py`. The request will fail if the token is invalid.
2.  **Sanitize Payload**: It calls `sanitize_incoming_payload` from `utils.py` to clean the input text and validate the request structure.
3.  **Extract Versions**: It uses `extract_versions_wst` to find all version numbers in the markdown text.
4.  **Split Markdown**: It calls `split_joined_markdown_text` to break the input markdown into separate chunks.
5.  **Single vs. Multi-File Logic**:
    *   **If `"End of Release Extract"` is NOT in `markdown_text`**: The application assumes a single document. It calls `generate_single_file_summary` from `utils.py` to create a concise summary and returns it immediately.
    *   **Otherwise**: It proceeds with the full multi-file analysis pipeline.
6.  **Extraction (Multi-File)**: It iterates through each `chunk` and `version`, initializing `Wst_MarkdownExtractor(chunk)` and calling its `.extract()` method.
7.  **Harmonization (Multi-File)**: It initializes `Wst_MarkdownHarmonizer()` and calls its `.harmonize()` method to create a single, cohesive document.
8.  **Crew Setup (Multi-File)**: It calls `setup_crew_wst` to configure the AI agents.
9.  **Crew Kickoff (Multi-File)**: It runs the crews sequentially, starting with the data crew, followed by the report and brief crews.
10. **Evaluation (Multi-File)**: It calls `evaluate_with_llm_judge` from `utils.py` to score the generated report.
11. **Response (Multi-File)**: It constructs and returns a `MultiFileAnalysisResponse` with all the generated data.

### Error Handling
- `ValidationError`: If the request body doesn't match the `MarkdownAnalysisRequest` model, it returns `422 Unprocessable Entity`.
- `Exception`: A general catch-all returns a `500 Internal Server Error`.

---

## 2. `utils.py`
This module provides a collection of utility functions that support the main application logic.

### `sanitize_incoming_payload(payload: dict) -> dict`
- **Purpose**: To clean and validate the raw JSON payload from the request.
- **Functionality**:
    - Validates the payload structure.
    - Cleans the `markdown_text` by removing or escaping problematic characters.
    - Validates that the `product` is one of the allowed values.
- **Returns**: A dictionary with the cleaned `markdown_text` and validated `product`.

### `extract_versions_wst(text)`
- **Purpose**: To find all version strings in a block of text.
- **Functionality**: Uses a regular expression to find version numbers (e.g., `XX.XX.XX.XX`).
- **Returns**: A sorted list of unique version strings.

### `split_joined_markdown_text(markdown_text: str) -> List[str]`
- **Purpose**: To split a string containing multiple documents into a list.
- **Functionality**: Splits the text using a `--- End of Release Extract ---` style separator.
- **Returns**: A list of strings, where each string is a markdown document.

### `async def generate_single_file_summary(markdown_text: str, product: str) -> str`
- **Purpose**: To generate a brief summary for a single markdown document.
- **Functionality**: Uses an `AzureChatOpenAI` LLM with a specific prompt to create a bullet-point summary.
- **Returns**: The content of the LLM's response as a string.

### `def evaluate_with_llm_judge(source_text: str, generated_report: str) -> dict`
- **Purpose**: To use an LLM to evaluate the quality of a generated report.
- **Functionality**: Uses an `AzureChatOpenAI` LLM to act as a "judge," scoring the report on accuracy, depth, and clarity, and then parses the score from the response.
- **Returns**: A dictionary containing the scores and the evaluation text.

### `def verify_auth_token(auth_token: str)`
- **Purpose**: To perform authentication by checking a token.
- **Functionality**: Compares the provided `auth_token` string against a hardcoded default token. If they do not match, it raises a `401 Unauthorized` HTTPException.

---

## 3. `wst_product_config.py`
This file is responsible for creating and configuring the CrewAI agents and tasks specifically for the "WST" product.

### `llm`
An instance of the `LLM` class, configured to use Azure OpenAI models. This instance is shared by all agents.

### `extract_json_from_output(raw_output: str) -> dict`
- **Purpose**: A helper function to safely extract a JSON object from an LLM's raw string output.

### `save_wst_metrics(output)`
- **Purpose**: A **callback** function executed after the `structurer_task`.
- **Functionality**: It extracts the JSON from the agent's output, performs some validation, and saves the final data to `shared_state.metrics`.

### `setup_crew_wst(extracted_text: str, versions: list)`
- **Purpose**: The main factory function that assembles and returns the different crews.

#### Agents & Tasks
- **`structurer` (Data Architect)**: Its task is to extract structured metrics from the markdown and format them into a canonical JSON object, as defined by the `STRUCTURER_PROMPT`. This task uses the `save_wst_metrics` callback.
- **`reporter` (Technical Writer)**: Its task is to take the structured data from the first agent and generate a **structured JSON report**, as defined by the detailed `REPORT_PROMPT`. The output is a JSON object, not a markdown document. This is then saved to `shared_state.report_parts["structured_report"]`.
- **`brief_writer` (Executive Summary Writer)**: Its task is to create a high-level, 3-5 bullet-point summary from the structured data. This is saved to `shared_state.report_parts["brief_summary"]`.

#### Crews
- **`data_crew`**: Contains the `structurer` agent. Runs first.
- **`report_crew`**: Contains the `reporter` agent. Depends on the `data_crew`.
- **`brief_summary_crew`**: Contains the `brief_writer` agent. Depends on the `data_crew`.

---

## 4. `models.py`
This file defines the Pydantic models used for data validation and serialization in the API.

- **`MarkdownAnalysisRequest`**: Defines the structure of the incoming request body. It requires `markdown_text` (string), `product` (string, "WST" or "TM"), and now **`auth`** (string) for the authentication token.
- **`SingleFileSummaryResponse`**: Defines the response for the single-file workflow.
- **`MultiFileAnalysisResponse`**: Defines the response for the multi-file workflow.

---

## 5. `shared_state.py`
This file provides a simple, thread-safe mechanism (`SharedState` class) for sharing data between the different CrewAI crews. A global `shared_state` instance is used to store `metrics` and `report_parts`.

---

## 6. `app_logging.py`
This file sets up the basic configuration for application-wide logging.

---

## 7. `wst__markdown_processor.py`
- **Status**: This file contains the core logic for processing markdown files.
- **Inferred Purpose**:
    - **`Wst_MarkdownExtractor`**: This class is responsible for parsing a single markdown chunk and extracting relevant, structured information from it.
    - **`Wst_MarkdownHarmonizer`**: This class takes the extracted data from multiple versions and combines them into a single, cohesive markdown document that can be fed to the AI agents.
