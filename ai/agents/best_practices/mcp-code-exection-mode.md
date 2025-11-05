
***

# Code Execution with MCP

## **Problem with Traditional MCP Tool Calls**

*   **Tool definitions** and **intermediate results** are stuffed into the LLM context.
*   Leads to:
    *   Huge **token usage**.
    *   **Latency** and **cost** spikes.
    *   **Privacy risks** (sensitive data flows through the model).
    *   **Copy/marshal errors** when passing large data between calls.
    *   Awkward handling of **loops, branching, retries**.

***

## **The Alternative: Code Execution Approach**

*   Represent MCP tools as **code APIs** (functions stored on disk).
*   Let the LLM **write executable code** that imports and composes these APIs.
*   Run the code in a **secure sandbox**.
*   Return only the **final result** to the model.

### **Benefits**

*   **Token efficiency**: No giant tool schemas or intermediate data in context.
*   **Privacy**: Sensitive data stays in sandbox; only summaries or small outputs return.
*   **Reliability**: Deterministic code handles data, reducing model errors.
*   **Complex workflows**: Natural with code (loops, conditions, error handling).
*   **Composable & auditable**: Scripts can be versioned, reused, and inspected.

***

## Example workflow
<img width="684" height="679" alt="image" src="https://github.com/user-attachments/assets/225cee02-a290-41ea-a52e-7e3b427e65a3" />


## **Python Analogy**

Instead of:

```python
# Traditional MCP: intermediate data flows through LLM
transcript = mcp.call("google-drive.getDocument", {"documentId": "abc123"})["content"]
mcp.call("salesforce.updateRecord", {"recordId": "XYZ", "data": {"Notes": transcript}})
```

Do:

```python
# Code Execution: intermediate data stays in sandbox
from servers.google_drive.get_document import get_document
from servers.salesforce.update_record import update_record

def run(document_id, record_id):
    transcript = get_document(document_id)["content"]
    update_record("SalesMeeting", record_id, {"Notes": transcript})
    return {"status": "ok"}
```

***

## **Common Theme**

*   Stop passing everything through the LLM.
*   Expose tools as **code APIs**.
*   Let the LLM **write and execute code** in a sandbox.
*   Return only what’s necessary.

***

## References
*  https://blog.cloudflare.com/code-mode/
*  https://www.anthropic.com/engineering/code-execution-with-mcp


