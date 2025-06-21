# API Documentation

## Overview
This document provides details about the FastAPI application for analyzing markdown files. The application exposes a single endpoint to process and analyze markdown content for different products.

## Authentication
The API uses a token-based authentication mechanism to secure its endpoints.

### How it Works
Authentication is handled by passing a token directly within the JSON payload of the request.

The server expects the `auth` field inside the main JSON object:
```json
{
  "markdown_text": "...",
  "product": "...",
  "auth": "<your_token>"
}
```

If the token is missing from the payload or invalid, the API will respond with a `401 Unauthorized` error, and the request will not be processed.

### Default Token
For development and testing purposes, a default authentication token is provided.

**Default Token**: `asdfghjkl123456788`

You must include this token in the `auth` field of your JSON payload to make successful API calls.

### Implementation Details
The authentication logic is implemented in the `verify_auth_token` function within the `utils.py` file. It is called directly from the `/analyze_markdown` endpoint and checks the `auth` field from the request model.

## Endpoints

### `/analyze_markdown`
This is the main endpoint for markdown analysis.

- **Method**: `POST`
- **Description**: Analyzes a given markdown text. The behavior changes based on the content of the markdown. If a single release note is provided, it returns a summary. If multiple, stitched-together release notes are provided, it performs a more complex analysis, harmonization, and evaluation.
- **Authentication**: Required. See the Authentication section above.
- **Request Body**: The endpoint expects a JSON body with the following structure:
    ```json
    {
      "markdown_text": "string",
      "product": "string",
      "auth": "string"
    }
    ```
    - `markdown_text`: A string containing the markdown content. This can be a single release note or multiple release notes "stitched" together using a separator like `--- End of Release Extract ---`.
    - `product`: A string indicating the product type. Supported values are `"WST"` or `"TM"`.
    - `auth`: The authentication token.

- **Responses**:
    - `200 OK`: Successful analysis. The response format depends on the input.
        - For a single release note, it returns a `SingleFileSummaryResponse` with a summary.
        - For multiple release notes, it returns a `MultiFileAnalysisResponse` containing metrics, a structured report, an AI-based evaluation, and a brief summary.
    - `401 Unauthorized`: If the authentication token is invalid or missing.
    - `422 Unprocessable Entity`: If the request body does not conform to the expected `MarkdownAnalysisRequest` schema.
    - `500 Internal Server Error`: For any other unexpected server-side errors during processing.

## How to Test the Endpoint

You can test the endpoint using a tool like `curl` or Postman.

**Example `curl` command:**

```bash
curl -X POST "http://127.0.0.1:8000/analyze_markdown" \
-H "Content-Type: application/json" \
-d '{
  "markdown_text": "## Release 45.1.0.0\n- New feature added.\n- Bug fix for critical issue.",
  "product": "WST",
  "auth": "asdfghjkl123456788"
}'
```

This command sends a `POST` request to the `/analyze_markdown` endpoint with a valid JSON payload that includes the `auth` token. 