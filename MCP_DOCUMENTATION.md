# Model Context Protocol (MCP) Implementation Documentation

## Table of Contents
1. [Overview](#overview)
2. [Code Location](#code-location)
3. [MCP Tools and Functionality](#mcp-tools-and-functionality)
4. [Tool Descriptions for AI Agents](#tool-descriptions-for-ai-agents)
5. [Data Models](#data-models)
6. [Mapping to REST API](#mapping-to-rest-api)
7. [Configuration and Deployment](#configuration-and-deployment)
8. [Usage Examples](#usage-examples)

---

## Overview

OLS4 provides a hosted Model Context Protocol (MCP) server that enables LLMs to access ontology terms and hierarchies through a standardized interface. The implementation uses Spring AI's MCP framework with the Streamable HTTP protocol (not legacy SSE).

**MCP Endpoint:** `{apiUrl}/api/mcp`
**Protocol:** Streamable HTTP
**Framework:** Spring AI 1.1.0-M2
**Reference:** https://modelcontextprotocol.io/

---

## Code Location

### Backend (Java/Spring Boot)

#### Model Classes
Location: `backend/src/main/java/uk/ac/ebi/spot/ols/model/mcp/`

- `McpClass.java` - Represents an ontology class with hierarchy information
- `McpEntityReference.java` - Lightweight reference to an entity
- `McpFetchResult.java` - OpenAI-compatible fetch result format
- `McpOntology.java` - Represents an ontology metadata
- `McpPage.java` - Generic pagination wrapper
- `McpSearchResult.java` - OpenAI-compatible search result format

#### Service Classes (MCP Tools)
Location: `backend/src/main/java/uk/ac/ebi/spot/ols/controller/mcp/`

- `McpClassService.java` - Exposes class search, ancestors, and descendants tools
- `McpOntologyService.java` - Exposes ontology listing tool
- `McpSearchService.java` - Exposes OpenAI-compatible search and fetch tools

#### Main Application
Location: `backend/src/main/java/uk/ac/ebi/spot/ols/Ols4Backend.java`

Tool registration via Spring beans:
```java
@Bean
public ToolCallbackProvider mcpOntologyTools(McpOntologyService service)
@Bean
public ToolCallbackProvider mcpClassTools(McpClassService service)
@Bean
public ToolCallbackProvider mcpSearchTools(McpSearchService service)
```

### Frontend (React/TypeScript)

- `frontend/src/pages/MCP.tsx` - MCP server information page
- Accessible via `/mcp` route in the UI

### Deployment

- `k8chart/ols4/templates/ols4-mcp-deployment.yaml` - Kubernetes deployment
- `k8chart/ols4/templates/ols4-mcp-service.yaml` - Kubernetes service

### Configuration

Location: `backend/src/main/resources/application.properties`

```properties
spring.ai.mcp.server.sse-endpoint=/api/mcp/sse
spring.ai.mcp.server.sse-message-endpoint=/api/mcp/message
spring.ai.mcp.server.protocol=STREAMABLE
spring.ai.mcp.server.streamable-http.mcp-endpoint=/api/mcp
```

---

## MCP Tools and Functionality

The MCP server exposes **6 tools** to AI agents, organized into 3 service categories:

### 1. McpOntologyService (1 tool)

#### `listOntologies`
**Description:** "Get all ontologies from OLS"

**Parameters:**
- `lang` (optional, default: "en") - Language code for labels

**Returns:** `List<McpOntology>`
- Each ontology includes: `ontologyId`, `label`, `definition`
- Limited to first 1000 ontologies

**Implementation:** Calls `OntologyRepository.find()` with pagination (page 0, size 1000)

---

### 2. McpClassService (3 tools)

#### `searchClasses`
**Description:** "Search all classes in OLS for a query string"

**Parameters:**
- `query` (required) - Search query string
- `ontologyId` (optional) - Restrict search to specific ontology
- `pageNum` (optional, default: 0) - Page number
- `pageSize` (optional, default: 20) - Results per page
- `lang` (optional, default: "en") - Language code

**Returns:** `McpPage<McpClass>` with pagination metadata
- Items: List of matching classes
- Metadata: `pageNum`, `pageSize`, `totalElements`, `totalPages`

**Features:**
- Full-text search across class labels and definitions
- Optional ontology filtering
- Pagination support
- Reference resolution enabled
- Manchester syntax enabled

**Implementation:** Calls `EntityRepository.find()` with type filter "class"

---

#### `getAncestors`
**Description:** "Get all ancestors for a class in OLS"

**Parameters:**
- `ontologyId` (required) - Ontology ID
- `classIri` (required) - IRI of the class
- `pageNum` (optional, default: 0) - Page number
- `pageSize` (optional, default: 20) - Results per page
- `lang` (optional, default: "en") - Language code

**Returns:** `McpPage<McpClass>` with pagination metadata

**Implementation:** Calls `ClassRepository.getAncestorsByOntologyId()`

---

#### `getDescendants`
**Description:** "Get all descendants of a class in OLS"

**Parameters:**
- `ontologyId` (required) - Ontology ID
- `classIri` (required) - IRI of the class
- `pageNum` (optional, default: 0) - Page number
- `pageSize` (optional, default: 20) - Results per page
- `lang` (optional, default: "en") - Language code

**Returns:** `McpPage<McpClass>` with pagination metadata

**Implementation:** Calls `ClassRepository.getDescendantsByOntologyId()`

---

### 3. McpSearchService (2 tools - OpenAI Compatible)

These tools are specifically designed to match OpenAI's MCP search/fetch pattern.

#### `search`
**Description:** "OpenAI compliant tool to search OLS for a query string"

**Parameters:**
- `query` (required) - Search query string

**Returns:** JSON string containing array of `McpSearchResult` objects

**McpSearchResult Format:**
```json
{
  "id": "ontologyId+entityIri",
  "title": "CURIE label",
  "url": "http://purl.obolibrary.org/obo/GO_0008150"
}
```

**Features:**
- Fixed pagination: page 0, size 20
- Language: always "en"
- Returns top 20 results as JSON string
- ID format: `{ontologyId}+{iri}` (e.g., "go+http://purl.obolibrary.org/obo/GO_0008150")

**Implementation:** Calls `EntityRepository.find()` and transforms to OpenAI format

---

#### `fetch`
**Description:** "OpenAI compliant tool to retrieve an entity from OLS by ID returned from the search tool. The ID must be of the format ontologyid+entityIri, e.g. go+http://purl.obolibrary.org/obo/GO_0008150. IDs in this format are returned by the OpenAI compliant 'search' tool."

**Parameters:**
- `id` (required) - Composite ID in format `{ontologyId}+{entityIri}`

**Returns:** JSON string containing `McpFetchResult` object

**McpFetchResult Format:**
```json
{
  "id": "ontologyId+entityIri",
  "title": "CURIE label",
  "text": "definition text",
  "url": "http://purl.obolibrary.org/obo/GO_0008150",
  "metadata": {
    "ontologyId": "go",
    "type": ["class"],
    "iri": "http://purl.obolibrary.org/obo/GO_0008150",
    "curie": "GO:0008150",
    "label": ["biological_process"],
    "definition": ["Any process..."],
    "directAncestor": [...],
    "directParent": [...],
    "hierarchicalParent": [...]
  }
}
```

**Features:**
- Parses composite ID (splits on '+')
- Validates ID format (throws exception if invalid)
- For classes: metadata contains full `McpClass` object
- For other types: metadata contains basic type information

**Implementation:** Calls `EntityRepository.getByOntologyIdAndIri()`

---

## Tool Descriptions for AI Agents

When an AI agent connects to the MCP server, it receives these tool definitions with their descriptions and parameter schemas. The descriptions are defined using Spring AI's `@Tool` and `@ToolParam` annotations.

### Tool Metadata Exposed to Agents

1. **listOntologies**
   - Purpose: Discover available ontologies
   - Use case: Agent needs to know what ontologies are available before searching
   - Description: "Get all ontologies from OLS"

2. **searchClasses**
   - Purpose: Find classes matching search criteria
   - Use case: Agent wants to find terms related to a concept
   - Description: "Search all classes in OLS for a query string"
   - Advanced: Supports ontology filtering and pagination

3. **getAncestors**
   - Purpose: Navigate class hierarchy upward
   - Use case: Agent needs to understand parent concepts
   - Description: "Get all ancestors for a class in OLS"

4. **getDescendants**
   - Purpose: Navigate class hierarchy downward
   - Use case: Agent needs to find more specific terms
   - Description: "Get all descendants of a class in OLS"

5. **search** (OpenAI compatible)
   - Purpose: Quick search for entities
   - Use case: OpenAI-style workflows, initial discovery
   - Description: "OpenAI compliant tool to search OLS for a query string"
   - Returns: Compact search results with IDs for fetching

6. **fetch** (OpenAI compatible)
   - Purpose: Retrieve full entity details
   - Use case: Get complete information after search
   - Description: "OpenAI compliant tool to retrieve an entity from OLS by ID returned from the search tool..."
   - Requires: ID from search results

### Tool Discovery Flow

An AI agent using the MCP server would typically:

1. Call `listOntologies()` to see available ontologies
2. Call `searchClasses(query="cancer")` or `search(query="cancer")` to find relevant terms
3. Call `getAncestors()` or `getDescendants()` to navigate hierarchy
4. Or: Call `fetch(id="mondo+http://...")` to get full details from search result

---

## Data Models

### McpClass
Represents an ontology class with hierarchy relationships.

```java
public class McpClass {
    public String ontologyId;           // e.g., "go", "efo", "mondo"
    public List<String> type;           // e.g., ["class"]
    public String iri;                  // Full IRI
    public String curie;                // Compact URI (e.g., "GO:0008150")
    public List<String> label;          // Labels in requested language
    public List<String> definition;     // Definitions in requested language
    public List<McpEntityReference> directAncestor;
    public List<McpEntityReference> directParent;
    public List<McpEntityReference> hierarchicalParent;
}
```

### McpEntityReference
Lightweight reference to another entity (used in relationships).

```java
public class McpEntityReference {
    public String iri;                  // Full IRI of referenced entity
    public List<String> definedBy;      // Ontology where defined
    public List<String> label;          // Label of referenced entity
}
```

### McpOntology
Represents an ontology's metadata.

```java
public class McpOntology {
    public String ontologyId;           // e.g., "go", "efo"
    public List<String> label;          // Ontology name
    public List<String> definition;     // Ontology description
}
```

### McpPage<T>
Generic pagination wrapper for results.

```java
public class McpPage<T> {
    public List<T> items;               // List of results
    public int pageNum;                 // Current page number (0-indexed)
    public int pageSize;                // Results per page
    public long totalElements;          // Total number of results
    public int totalPages;              // Total number of pages
}
```

### McpSearchResult
OpenAI-compatible search result format.

```java
public class McpSearchResult {
    public String id;                   // Format: "{ontologyId}+{iri}"
    public String title;                // Format: "{curie} {label}"
    public String url;                  // Entity IRI
}
```

### McpFetchResult
OpenAI-compatible fetch result format.

```java
public class McpFetchResult {
    public String id;                   // Format: "{ontologyId}+{iri}"
    public String title;                // Format: "{curie} {label}"
    public String text;                 // Entity definition
    public String url;                  // Entity IRI
    public Object metadata;             // Full McpClass for classes, type info for others
}
```

---

## Mapping to REST API

The MCP tools are thin wrappers around existing OLS REST API functionality. Here's how they map:

### 1. MCP `listOntologies()` → REST API

**MCP Tool:**
```
listOntologies(lang="en")
```

**Equivalent REST API Call:**
```
GET /api/v2/ontologies?page=0&size=1000&lang=en
```

**Controller:** `V2OntologyController.getOntologies()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2OntologyController.java:40`

**Differences:**
- MCP: Fixed page size of 1000, returns list directly
- REST: Configurable pagination, returns `V2PagedAndFacetedResponse`
- MCP: Returns simplified `McpOntology` model
- REST: Returns full `V2Entity` model with all fields

---

### 2. MCP `searchClasses()` → REST API

**MCP Tool:**
```
searchClasses(
  query="cancer",
  ontologyId="mondo",
  pageNum=0,
  pageSize=20,
  lang="en"
)
```

**Equivalent REST API Call:**
```
GET /api/v2/entities?search=cancer&type=class&ontologyId=mondo&page=0&size=20&lang=en
```

**Controller:** `V2EntityController.getEntities()` (with type=class filter)
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2EntityController.java:42`

**Differences:**
- MCP: Type filter "class" is hardcoded in the service
- REST: Type must be passed as query parameter or searchProperty
- MCP: Returns `McpPage<McpClass>` with simplified model
- REST: Returns `V2PagedAndFacetedResponse<V2Entity>` with full model
- MCP: Always enables `resolveReferences` and `manchesterSyntax`
- REST: These are optional parameters

**Alternative REST Endpoint:**
```
GET /api/v2/classes?search=cancer&page=0&size=20&lang=en
```

**Controller:** `V2ClassController.getClasses()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2ClassController.java:85`

---

### 3. MCP `getAncestors()` → REST API

**MCP Tool:**
```
getAncestors(
  ontologyId="go",
  classIri="http://purl.obolibrary.org/obo/GO_0008150",
  pageNum=0,
  pageSize=20,
  lang="en"
)
```

**Equivalent REST API Call:**
```
GET /api/v2/ontologies/go/classes/http%253A%252F%252Fpurl.obolibrary.org%252Fobo%252FGO_0008150/ancestors?page=0&size=20&lang=en
```

**Controller:** `V2ClassController.getAncestorsByOntology()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2ClassController.java:277`

**Differences:**
- MCP: IRI passed as plain string parameter
- REST: IRI must be double URL encoded in path
- MCP: Always excludes obsolete entities (includeObsoleteEntities=false)
- REST: `includeObsoleteEntities` is a configurable parameter (default: false)
- MCP: Returns `McpPage<McpClass>`
- REST: Returns `V2PagedResponse<V2Entity>`

---

### 4. MCP `getDescendants()` → REST API

**MCP Tool:**
```
getDescendants(
  ontologyId="go",
  classIri="http://purl.obolibrary.org/obo/GO_0008150",
  pageNum=0,
  pageSize=20,
  lang="en"
)
```

**Equivalent REST API Call:**
```
GET /api/v2/ontologies/go/classes/http%253A%252F%252Fpurl.obolibrary.org%252Fobo%252FGO_0008150/descendants?page=0&size=20&lang=en
```

**Controller:** `V2ClassController.getDescendantsByOntology()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2ClassController.java:309`

**Differences:**
- Same as `getAncestors()` mapping

---

### 5. MCP `search()` → REST API

**MCP Tool:**
```
search(query="cancer")
```

**Equivalent REST API Call:**
```
GET /api/v2/entities?search=cancer&page=0&size=20&lang=en
```

**Controller:** `V2EntityController.getEntities()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2EntityController.java:42`

**Differences:**
- MCP: Fixed pagination (page 0, size 20), fixed language (en)
- REST: Fully configurable pagination and language
- MCP: Returns JSON string with `McpSearchResult[]` format (OpenAI compatible)
- REST: Returns `V2PagedAndFacetedResponse<V2Entity>` with full metadata
- MCP: Simplified result with id, title, url only
- REST: Full entity data with all fields
- MCP: ID format is composite: `{ontologyId}+{iri}`
- REST: Entities returned with separate ontologyId and iri fields

---

### 6. MCP `fetch()` → REST API

**MCP Tool:**
```
fetch(id="go+http://purl.obolibrary.org/obo/GO_0008150")
```

**Equivalent REST API Call:**
```
GET /api/v2/ontologies/go/entities/http%253A%252F%252Fpurl.obolibrary.org%252Fobo%252FGO_0008150?lang=en
```

**Controller:** `V2EntityController.getEntity()`
**File:** `backend/src/main/java/uk/ac/ebi/spot/ols/controller/api/v2/V2EntityController.java:158`

**Differences:**
- MCP: Takes composite ID, parses into ontologyId + iri
- REST: Takes ontologyId and iri as separate path parameters
- MCP: Returns JSON string with `McpFetchResult` format (OpenAI compatible)
- REST: Returns `V2Entity` object directly
- MCP: Wraps entity in fetch result with id, title, text, url, metadata
- REST: Returns entity with all fields directly
- MCP: Fixed language (en)
- REST: Configurable language parameter

---

## Configuration and Deployment

### Spring Boot Configuration

**File:** `backend/src/main/resources/application.properties`

```properties
# MCP Server Configuration
spring.ai.mcp.server.sse-endpoint=/api/mcp/sse
spring.ai.mcp.server.sse-message-endpoint=/api/mcp/message
spring.ai.mcp.server.protocol=STREAMABLE
spring.ai.mcp.server.streamable-http.mcp-endpoint=/api/mcp
```

### Dependencies

**File:** `backend/build.gradle` (implied from code)

```groovy
implementation 'org.springframework.ai:spring-ai-starter-mcp-server-webmvc:1.1.0-M2'
```

### Kubernetes Deployment

**Deployment File:** `k8chart/ols4/templates/ols4-mcp-deployment.yaml`
- Uses same container as main OLS backend
- Resource limits: 10Gi memory request, 50Gi limit
- CPU: 0.5 request, 2 limit
- Health checks: `/ols4/api/v2/health`

**Service File:** `k8chart/ols4/templates/ols4-mcp-service.yaml`
- Separate Kubernetes service for MCP endpoint
- Routes to main OLS backend pods

### Frontend Integration

**Page:** `frontend/src/pages/MCP.tsx`
- Route: `/mcp`
- Displays MCP endpoint URL
- Notes protocol type (Streamable HTTP)
- Links to MCP documentation

---

## Usage Examples

### Example 1: Listing Available Ontologies

**MCP Tool Call:**
```json
{
  "tool": "listOntologies",
  "parameters": {
    "lang": "en"
  }
}
```

**Response:**
```json
[
  {
    "ontologyId": "go",
    "label": ["Gene Ontology"],
    "definition": ["The Gene Ontology (GO) project provides..."]
  },
  {
    "ontologyId": "efo",
    "label": ["Experimental Factor Ontology"],
    "definition": ["The Experimental Factor Ontology (EFO) provides..."]
  }
]
```

---

### Example 2: Searching for Classes

**MCP Tool Call:**
```json
{
  "tool": "searchClasses",
  "parameters": {
    "query": "heart disease",
    "ontologyId": "mondo",
    "pageSize": 5
  }
}
```

**Response:**
```json
{
  "items": [
    {
      "ontologyId": "mondo",
      "type": ["class"],
      "iri": "http://purl.obolibrary.org/obo/MONDO_0005267",
      "curie": "MONDO:0005267",
      "label": ["heart disease"],
      "definition": ["A disease that involves the heart."],
      "directAncestor": [
        {
          "iri": "http://purl.obolibrary.org/obo/MONDO_0005385",
          "label": ["cardiovascular disorder"]
        }
      ]
    }
  ],
  "pageNum": 0,
  "pageSize": 5,
  "totalElements": 42,
  "totalPages": 9
}
```

---

### Example 3: Getting Class Ancestors

**MCP Tool Call:**
```json
{
  "tool": "getAncestors",
  "parameters": {
    "ontologyId": "mondo",
    "classIri": "http://purl.obolibrary.org/obo/MONDO_0005267",
    "pageSize": 10
  }
}
```

**Response:**
```json
{
  "items": [
    {
      "ontologyId": "mondo",
      "curie": "MONDO:0005385",
      "label": ["cardiovascular disorder"],
      "directParent": [...]
    },
    {
      "ontologyId": "mondo",
      "curie": "MONDO:0000001",
      "label": ["disease"],
      "directParent": [...]
    }
  ],
  "pageNum": 0,
  "pageSize": 10,
  "totalElements": 4,
  "totalPages": 1
}
```

---

### Example 4: OpenAI-Compatible Search and Fetch

**Step 1: Search**
```json
{
  "tool": "search",
  "parameters": {
    "query": "diabetes"
  }
}
```

**Response (JSON string):**
```json
"[{\"id\":\"mondo+http://purl.obolibrary.org/obo/MONDO_0005015\",\"title\":\"MONDO:0005015 diabetes mellitus\",\"url\":\"http://purl.obolibrary.org/obo/MONDO_0005015\"},{\"id\":\"mondo+http://purl.obolibrary.org/obo/MONDO_0005148\",\"title\":\"MONDO:0005148 type 2 diabetes mellitus\",\"url\":\"http://purl.obolibrary.org/obo/MONDO_0005148\"}]"
```

**Step 2: Fetch Details**
```json
{
  "tool": "fetch",
  "parameters": {
    "id": "mondo+http://purl.obolibrary.org/obo/MONDO_0005015"
  }
}
```

**Response (JSON string):**
```json
"{\"id\":\"mondo+http://purl.obolibrary.org/obo/MONDO_0005015\",\"title\":\"MONDO:0005015 diabetes mellitus\",\"text\":\"A metabolic disorder characterized by abnormally high blood sugar levels...\",\"url\":\"http://purl.obolibrary.org/obo/MONDO_0005015\",\"metadata\":{\"ontologyId\":\"mondo\",\"type\":[\"class\"],\"iri\":\"http://purl.obolibrary.org/obo/MONDO_0005015\",\"curie\":\"MONDO:0005015\",\"label\":[\"diabetes mellitus\"],\"definition\":[\"A metabolic disorder...\"],\"directAncestor\":[...],\"directParent\":[...],\"hierarchicalParent\":[...]}}"
```

---

## Key Technical Details

### Framework and Architecture
- **Framework:** Spring AI 1.1.0-M2
- **Dependency:** `spring-ai-starter-mcp-server-webmvc`
- **Tool Registration:** Declarative using `@Tool` and `@ToolParam` annotations
- **Provider:** `MethodToolCallbackProvider` for registering service methods as tools
- **Protocol:** Streamable HTTP (modern MCP protocol, not legacy SSE)

### Data Processing
- **JSON Processing:** Google Gson for serialization/deserialization
- **Database:** Neo4j (graph database) and Solr (search engine)
- **Transform Options:**
  - `resolveReferences = true` - Resolve entity references
  - `manchesterSyntax = true` - Use Manchester syntax for complex expressions

### Repository Layer
All MCP services delegate to existing OLS repository layer:
- `EntityRepository` - Generic entity queries and search
- `ClassRepository` - Class-specific queries (ancestors, descendants)
- `OntologyRepository` - Ontology metadata queries

### OpenAI Compatibility
The `McpSearchService` specifically implements OpenAI's search/fetch pattern:
- **search:** Returns lightweight results with IDs
- **fetch:** Takes ID from search results, returns full details
- **ID Format:** `{ontologyId}+{iri}` (composite key)
- **Reference:** https://platform.openai.com/docs/mcp#create-an-mcp-server

### Language Support
- All tools support language parameter
- Default: "en" (English)
- Affects labels, definitions, and other multilingual fields
- Language codes follow standard ISO 639-1 format

### Pagination
- Default page size: 20 (configurable except in OpenAI tools)
- Default page number: 0 (zero-indexed)
- Pagination metadata included in `McpPage` responses
- OpenAI tools (`search`, `fetch`) use fixed pagination

### Security and Validation
- ID validation in `fetch()` tool ensures correct format
- URL decoding handled automatically by REST controllers
- No authentication/authorization in MCP layer (delegated to infrastructure)

---

## Summary

The OLS4 MCP implementation provides:

1. **6 specialized tools** for ontology access via AI agents
2. **Thin wrappers** around existing REST API functionality
3. **OpenAI compatibility** via dedicated search/fetch tools
4. **Simplified data models** optimized for AI consumption
5. **Production-ready deployment** via Kubernetes with health checks
6. **Standard MCP protocol** (Streamable HTTP) for broad client compatibility

The MCP server enables LLMs to:
- Discover available ontologies
- Search for ontology terms
- Navigate class hierarchies (ancestors/descendants)
- Retrieve full entity details
- Use familiar OpenAI search/fetch patterns

All functionality is backed by OLS's existing robust REST API, Neo4j graph database, and Solr search infrastructure.
