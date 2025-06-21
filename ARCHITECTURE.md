# Application Architecture Diagram

This diagram illustrates the complete workflow of the FastAPI application, from receiving a client request to sending the final response. It covers both the single-file summary path and the multi-file analysis path.

```mermaid
graph TD;
    A[Client] -- "POST /analyze_markdown<br/>(markdown, product, auth_token)" --> B[main.py: FastAPI Endpoint];

    subgraph "1. Initial Processing"
        B -- "Calls verify_auth_token(request.auth)" --> U1(utils.py);
        U1 -- "Sanitizes input" --> U2(utils.py: sanitize_incoming_payload);
        U2 -- "Splits markdown" --> U3(utils.py: split_joined_markdown_text);
        U3 -- "Returns text parts" --> B;
    end

    B --> D{Is it a multi-part document?};

    subgraph "Path A: Single-File Summary"
        D -- "No" --> S1[utils.py: generate_single_file_summary];
        S1 -- "Returns simple summary" --> B;
    end

    subgraph "Path B: Multi-File Analysis"
        D -- "Yes" --> M1[UnifiedMarkdownExtractor];
        M1 -- "Extracts content" --> M2[UnifiedMarkdownHarmonizer];
        M2 -- "Harmonizes text" --> M3[wst_product_config.py];
        M3 -- "Sets up AI Crews" --> M4["CrewAI Agents<br/>(Data, Report, Brief)"];
        M4 -- "Store results" --> M5[shared_state.py];
        B -- "Reads state & evaluates" --> M6[utils.py: evaluate_with_llm_judge];
        M6 -- "Returns evaluation score" --> B;
    end

    B -- "Aggregates results and builds response" --> R[JSON Response];
    R --> A;
``` 