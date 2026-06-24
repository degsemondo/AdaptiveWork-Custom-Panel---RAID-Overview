# Data Field - v2.0

```json
{
  "sessionId": {{GetSessionId()}},
  "project": {{JsonObject(CurrentObject(),"id")}}
}
```

## Field Explanations

### sessionId
- **Source**: `{{GetSessionId()}}`
- **Purpose**: Obtains the current user's session ID for API authentication
- **Used in**: REST API calls with `Authorization: Session {sessionId}` header

### project
- **Source**: `{{JsonObject(CurrentObject(),"id")}}`
- **Purpose**: Gets the current Project object's full ID path (e.g., `/Project/xyz`)
- **Used in**: CZQL WHERE clause to filter Risks, Issues, and Requests for this project
- **Example Query**: `SELECT * FROM Risk WHERE PlannedFor = '{projectId}'`

## Notes
- These Planview template variables are evaluated when the panel loads
- The sessionId is passed as a Session header in fetch requests
- The project ID is used in all CZQL queries to scope data to current project
