# Data Field - v1.0 Working Version

```json
{
  "sessionId": {{GetSessionId()}},
  "project": {{JsonObject(CurrentObject(),"SYSID")}}
}
```

## Field Explanations

### sessionId
- **Source**: `{{GetSessionId()}}`
- **Purpose**: Obtains the current user's session ID for API authentication
- **Used in**: CZQL queries and API calls for authorization header

### project
- **Source**: `{{JsonObject(CurrentObject(),"SYSID")}}`
- **Purpose**: Gets the current Project object's SYSID (system ID)
- **Used in**: CZQL WHERE clause to filter Risks, Issues, and Requests for this project
- **Example Query**: `SELECT * FROM Risk WHERE PlannedFor = '{projectId}'`

## Notes
- These Planview template variables are evaluated when the panel loads
- The sessionId is passed as Bearer token in fetch headers
- The project SYSID is used in all CZQL queries to scope data to current project
