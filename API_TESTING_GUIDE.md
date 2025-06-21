# API Testing Guide

This guide explains how to create a valid JSON payload for testing the `/analyze_markdown` endpoint and how to use it to make a request.

---

## 1. Generating the Payload with Python

You can dynamically create the payload for the API by reading a markdown file from your local system. The following Python script demonstrates how to do this.

### Script

```python
import json
import os

# 1. Define the path to your markdown file
file_path = "path/to/your/markdown.md" # Replace with the actual file path

# 2. Define product and auth token
product = "WST"
auth_token = os.getenv("API_AUTH_TOKEN", "asdfghjkl123456788") # Use env var or default

# 3. Read markdown file
try:
    with open(file_path, "r", encoding="utf-8") as f:
        md = f.read()
except FileNotFoundError:
    print(f"Error: The file '{file_path}' was not found.")
    exit(1)

# 4. Construct the payload
payload = {
    "markdown_text": md,
    "product": product,
    "auth": auth_token
}

# 5. Print the payload as a JSON string
print(json.dumps(payload, indent=2))
```

### How it Works
1.  **Define Paths & Config**: Set the `file_path` to your markdown file. The `auth_token` is taken from an environment variable `API_AUTH_TOKEN` or uses the default if not set.
2.  **Read File**: The script reads the content of your markdown file.
3.  **Construct Payload**: It creates a Python dictionary named `payload` with the required keys: `markdown_text`, `product`, and crucially, `auth`.
4.  **Print JSON**: It converts the dictionary into a JSON formatted string and prints it to the console.

---

## 2. Testing the Endpoint with `curl`

You can use the Python script to generate the payload and pipe it directly into `curl`.

### Authentication Note
The authentication token is now sent **inside the JSON payload**, not as a separate `Authorization` header.

### Steps to Test
1.  **Save the script**: Save the Python code above into a file (e.g., `create_payload.py`).
2.  **(Optional) Set Environment Variable**: `export API_AUTH_TOKEN="your_real_token"`
3.  **Run and pipe to `curl`**: Execute the script and send its output to the `curl` command.

### Example `curl` Command
This command runs the script and uses its output as the request body (`-d @-`).

```bash
python create_payload.py | curl -X POST "http://127.0.0.1:8000/analyze_markdown" \
-H "Content-Type: application/json" \
-d @-
```
- `python create_payload.py`: Runs the script to generate and print the JSON payload.
- `|`: Pipes the JSON output to `curl`.
- `curl ...`: Sends the POST request.
- `-d @-`: Tells `curl` to read the request data from standard input (what it's receiving from the pipe). 