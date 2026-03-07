# Streaming Flow Diagram

End-to-end sequence of a user message through the Claude Agent web client stack.

```mermaid
sequenceDiagram
    actor User
    participant UI as React UI<br/>(useClaudeAgent.js)
    participant API as API Client<br/>(client.js)
    participant BE as Backend<br/>(messages.py / invocations.py)
    participant SM as SessionManager
    participant SDK as Claude Agent SDK

    %% Session creation (lazy, on first message)
    User->>UI: Types message & hits send
    alt No session yet
        UI->>API: createSession({model, cwd, mcp_server_ids})
        API->>BE: POST /sessions
        BE->>SM: create AgentSession
        SM->>SDK: client.connect()
        SDK-->>SM: connected
        BE-->>API: {session_id}
        API-->>UI: session_id stored
    end

    %% Message send
    UI->>API: sendMessageStream(session_id, message)
    alt Direct mode
        API->>BE: POST /sessions/{id}/messages/stream
    else Invocations mode
        API->>BE: POST /invocations<br/>{path: "/sessions/{id}/messages/stream"}
    end

    BE->>SM: get_or_ensure_session()
    SM-->>BE: AgentSession
    BE->>SDK: session.send_message_stream(message)

    %% SSE stream opens — kept alive
    BE-->>API: SSE stream opened (text/event-stream)

    SDK-->>BE: data: {type: "start"}
    BE-->>API: data: {type: "start"}
    API-->>UI: onmessage → update session_id if changed

    loop Claude is thinking / using tools
        SDK-->>BE: AssistantMessage (TextBlock)
        BE-->>API: data: {type: "text", content: "..."}
        API-->>UI: onmessage → append/stream text to chat

        SDK-->>BE: AssistantMessage (ToolUseBlock)
        BE-->>API: data: {type: "tool_use", tool_name, tool_input}
        API-->>UI: onmessage → show tool card in chat

        alt Tool needs permission
            SDK->>BE: permission_callback(tool_name)
            BE->>BE: permission_queue.put(request)
            BE-->>API: data: {type: "permission", permission: {...}}
            API-->>UI: onmessage → handlePermissionEvent()
            UI-->>User: Show Allow/Deny dialog

            User->>UI: Clicks Allow or Deny
            UI->>API: respondToPermission(request_id, allowed)
            API->>BE: POST /sessions/{id}/permissions/respond
            BE->>SDK: permission_event.set() → PermissionResultAllow/Deny
            Note over SDK,BE: SDK resumes execution
        end
    end

    SDK-->>BE: ResultMessage (cost, num_turns, session_id)
    BE-->>API: data: {type: "result", cost_usd, session_id}
    API-->>UI: onmessage → update session_id, show cost

    BE-->>API: data: {type: "done", session_id}
    API-->>UI: onmessage → eventSource.close()
    UI-->>User: Final message rendered, stream closed
```

## Key behaviours

- **Session is created lazily** — only on the first message if none exists
- **SSE stream stays open** for the entire duration of Claude's response, including multiple tool calls
- **Permission is an interrupt** — the stream keeps the connection alive while waiting for user input via a separate POST
- **Session ID can change** — the SDK assigns a real ID on first use; the frontend watches `start`, `result`, and `done` events to update it
