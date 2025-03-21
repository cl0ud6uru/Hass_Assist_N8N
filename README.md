
## **📖 N8N Integration with Home Assistant Assist**
This guide explains how to **integrate N8N with Home Assistant Assist** using the **Ollama integration** to use N8N as the Conversation agent.
(This readme was written by ChatGPT for now because Im lazy)

![image](https://github.com/user-attachments/assets/bb25c8f3-2baf-4c8d-bee8-002ad3550ad3)


---

## **🚀 Features**
✅ **Use N8N as an AI processing backend** for Home Assistant Assist.  
✅ **Seamlessly integrate with external AI models** (Groq, OpenAI, etc.).  
✅ **Trick the Ollama integration** into treating N8N as an AI model.  
✅ **Process AI responses and send them back in Ollama-compatible JSON format**.

---

## **📌 1. Setting Up N8N with Docker & Portainer**
We will deploy **N8N using Docker and Portainer** with the following stack.

### **🛠 N8N Docker Stack Configuration**
Create a new **Stack** in Portainer and use the following configuration:

```yaml
version: "2.1"
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=n8n.yourdomain.com
      - N8N_ENDPOINT_WEBHOOK=apitest     
      - N8N_ENDPOINT_WEBHOOK_TEST=api
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

> **Note:**  
> If you want to use **Test Mode in N8N**, set `N8N_ENDPOINT_WEBHOOK_TEST=api`.  
> However, you **cannot** use `"api"` for both `N8N_ENDPOINT_WEBHOOK_TEST` and `N8N_ENDPOINT_WEBHOOK`.  
> To switch between **test** and **production**, change `N8N_ENDPOINT_WEBHOOK` to `"apitest"`.

---

## **🛠 2. Configuring N8N Workflow**
Once N8N is running, you will need to **create the following nodes**.

### **📌 Step 1: Create Webhook Nodes**
These **first two webhooks** trick Home Assistant’s **Ollama integration** into accepting N8N.

#### **🌐 Webhook 1: "GET Version"**
| Setting        | Value |
|---------------|----------------|
| **Node Type** | Webhook |
| **Name**      | `GET Version` |
| **Method**    | `GET` |
| **Path**      | `/version` |
| **Authentication** | `None` |
| **Respond Immediately** | ✅ |
| **Response Data** |  
```json
{ "version": "0.5.13" }
```

![image](https://github.com/user-attachments/assets/e56f15d0-29e0-40d7-9a07-81bb2b642a04)

#### **🌐 Webhook 2: "List Models"**
| Setting        | Value |
|---------------|----------------|
| **Node Type** | Webhook |
| **Name**      | `List Models` |
| **Method**    | `GET` |
| **Path**      | `/tags` |
| **Authentication** | `None` |
| **Respond Immediately** | ✅ |
| **Response Data** |
```json
{
  "models": [
    { "name": "llama3.2:latest", "model": "llama3.2:latest" },
    { "name": "deepseek-r1:latest", "model": "deepseek-r1:latest" }
  ]
}
```

![image](https://github.com/user-attachments/assets/e734ed8d-d91b-4a91-9d99-30bd0310b78e)

✅ **These allow you to add N8N to Home Assistant as if it were an Ollama server.**

---

### **📌 Step 2: Create Chat and Response Webhooks**
These webhooks **handle actual AI queries from Assist**.

#### **📝 Webhook 3: "Chat" (Receives Requests)**
| Setting        | Value |
|---------------|----------------|
| **Node Type** | Webhook |
| **Name**      | `Chat` |
| **Method**    | `POST` |
| **Path**      | `/chat` |
| **Authentication** | `None` |
| **Respond Using** | `Respond to Webhook Node` |

![image](https://github.com/user-attachments/assets/b0081dd9-7042-45fb-b4d9-6658f8e5cba2)

---

#### **🤖 AI Agent Node**
| Setting | Value |
|---------------|----------------|
| **Node Type** | `AI Agent` |
| **Prompt (User Message)** |  
```json
{{ $json.body.messages[$json.body.messages.length - 1].content }}
```

✅ **Ensures the AI always processes the latest user input.**

![image](https://github.com/user-attachments/assets/776dfa29-8b12-4135-b8bb-d7d52288b674)



---

#### **🧪 Set Node: “Edit Fields” (Fixes JSON for Webhook)**
This node ensures that special characters like newlines and quotes are escaped properly before sending to Home Assistant.

| Setting | Value |
|--------|-------|
| **Node Type** | Set |
| **Name** | `Edit Fields` |
| **Field Name** | `output` |
| **Value (Expression)** |
```javascript
{{ $json.output.replace(/\\n/g, "\\\\n").replace(/"/g, '\\"') }}
```
![image](https://github.com/user-attachments/assets/81d22cd5-2381-46d8-9659-8bd3468e79a2)

✅ **This prevents JSON parsing issues in the Webhook response.**

---

#### **🔁 Respond to Webhook Node**
| Setting | Value |
|---------------|----------------|
| **Node Type** | `Respond to Webhook` |
| **Respond With** | JSON |
| **Response Body** |
```json
{
  "model": "llama3.2:latest",
  "created_at": "{{ $now }}",
  "message": {
    "role": "assistant",
    "content": "{{ $json.output }}"
  },
  "done": true
}
```
![image](https://github.com/user-attachments/assets/f7a7b6af-e42e-4886-8586-67dd7cb2eeff)

---

## **🔗 3. Configuring Home Assistant**
Once N8N is properly set up, you need to **connect it to Home Assistant**.

### **📌 Install and Configure Ollama Integration**
1. In **Home Assistant**, go to **Settings → Integrations**.
2. Click **Add Integration** → Search for `Ollama`.
3. Enter the URL of your N8N instance:
   ```
   https://n8n.yourdomain.com
   ```
4. Save and restart Home Assistant.

✅ **Assist will now route queries to N8N!**

---

## **🚀 4. Final Testing**
### **Steps to Test**
1. Open **Assist** in Home Assistant.
2. Ask a simple question:
   ```
   What is the weather today?
   ```
3. Check N8N logs:
   - The **Chat webhook** should receive the request.
   - The **AI Agent node** should process the response.
   - The **Set node** ensures formatting is clean.
   - The **Webhook Response node** should send back a valid JSON reply.

4. If responses are not appearing:
   - Check **Home Assistant logs** (`Developer Tools → Logs`).
   - Run a **manual webhook test** using `curl`:
     ```bash
     curl -X POST "https://n8n.yourdomain.com/chat" \
          -H "Content-Type: application/json" \
          -d '{"model": "llama3.2:latest", "messages": [{"role": "user", "content": "Hello"}]}'
     ```

---

## **📌 Summary**
✅ **Tricked Home Assistant’s Ollama Integration** into accepting N8N as a model.  
✅ **Intercepted Assist requests** and forwarded them to an external AI.  
✅ **Formatted the response correctly using a Set Node**.  
✅ **Successfully integrated Home Assistant Assist with N8N for custom AI workflows!**  

---

### **🎯 Next Steps**
- **Expand the AI workflow** by integrating **Google Calendar, smart home controls, or external APIs**.
- **Optimize AI responses** by tweaking the prompt in **AI Agent Node**.
- **Improve latency** by enabling **streaming responses** (`"stream": true`).

---
