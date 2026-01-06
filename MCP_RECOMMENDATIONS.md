# MCP Enhancement Recommendations for OLS4

## Executive Summary

The current OLS4 MCP implementation relies heavily on agents' latent knowledge of ontologies. Based on MCP best practices and specifications, we recommend adding **Prompts**, **Resources**, and **Enhanced Tool Descriptions** to significantly improve agent effectiveness and user experience.

## Priority 1: Add Prompt Templates

Prompts are predefined instruction templates that standardize how agents perform common tasks. [MCP Prompts Documentation](https://modelcontextprotocol.io/specification/2025-06-18/server/prompts)

### Recommended Prompts

#### 1. `explore_concept_hierarchy`
**Purpose:** Guide agents through exploring a biological/medical concept and its relationships

**Template:**
```
You are helping explore the concept: {{concept}}

Follow this workflow:
1. Use searchClasses(query="{{concept}}") to find relevant terms
2. Review results and identify the most appropriate class
3. Use getAncestors() to understand parent concepts
4. Use getDescendants() to find more specific related terms
5. Summarize the hierarchical context

When interpreting results:
- IRI: Unique identifier for the concept (e.g., http://purl.obolibrary.org/obo/GO_0008150)
- CURIE: Compact form (e.g., GO:0008150)
- directParent: Immediate parent in hierarchy
- directAncestor: All parents up the hierarchy

Recommend which ontology seems most appropriate based on the domain:
- MONDO: Diseases and disorders
- GO: Biological processes, molecular functions, cellular components
- EFO: Experimental variables and conditions
- CHEBI: Chemical compounds and entities
- HP: Human phenotypes
```

**Variables:** `concept` (string)

---

#### 2. `find_disease_hierarchy`
**Purpose:** Specialized prompt for disease-related searches

**Template:**
```
You are searching for disease terms related to: {{disease_query}}

Recommended workflow:
1. Search in MONDO ontology first: searchClasses(query="{{disease_query}}", ontologyId="mondo")
2. If no results, try broader search: searchClasses(query="{{disease_query}}")
3. For each relevant result:
   - Note the CURIE identifier (e.g., MONDO:0005015)
   - Check ancestors to understand disease classification
   - Check descendants to find more specific subtypes

MONDO ontology covers:
- Diseases and medical conditions
- Inherited disorders
- Infectious diseases
- Neoplasms and cancers
- Syndromes

Related ontologies to consider:
- HP: Human Phenotype Ontology (symptoms/phenotypes)
- DOID: Disease Ontology (alternative disease classifications)
```

**Variables:** `disease_query` (string)

---

#### 3. `compare_terms_across_ontologies`
**Purpose:** Help agents compare how different ontologies represent similar concepts

**Template:**
```
You are comparing representations of: {{term}}

Process:
1. First, list available ontologies: listOntologies()
2. Search across all ontologies: searchClasses(query="{{term}}")
3. Group results by ontologyId
4. For each ontology's representation:
   - Note the preferred label
   - Compare definitions
   - Examine hierarchy placement
   - Note any cross-references

Different ontologies may model the same concept differently:
- GO focuses on biological processes and functions
- MONDO focuses on disease classification
- EFO focuses on experimental and measurement contexts
- Understanding these perspectives helps choose the right ontology for your use case

Report differences in:
- Scope (how broadly/narrowly defined)
- Hierarchy (parent concepts)
- Definition (explanatory text)
- Cross-references (links to other ontologies)
```

**Variables:** `term` (string)

---

#### 4. `openai_search_and_fetch_workflow`
**Purpose:** Guide agents using OpenAI-compatible tools

**Template:**
```
You are using OpenAI-compatible MCP tools to search OLS for: {{query}}

Step 1 - Search:
Call: search(query="{{query}}")
Returns: Array of results with id, title, url

Step 2 - Analyze results:
Each result has a composite ID format: ontologyId+entityIri
Example: "mondo+http://purl.obolibrary.org/obo/MONDO_0005015"

Step 3 - Fetch details:
For interesting results, call: fetch(id="[id from search results]")
Returns: Full entity with metadata including:
- Complete definition
- Hierarchical relationships (parents, ancestors)
- CURIE identifier
- Source ontology

Important notes:
- search() returns max 20 results
- fetch() requires exact ID from search results
- metadata field contains full ontology class structure
- Use IRI from url field to link to web interface
```

**Variables:** `query` (string)

---

#### 5. `navigate_ontology_tree`
**Purpose:** Guide agents in traversing ontology hierarchies

**Template:**
```
You are navigating the ontology tree starting from: {{starting_concept}}
Direction: {{direction}} (up to ancestors or down to descendants)

Workflow for navigating UP (to more general concepts):
1. Find the starting class: searchClasses(query="{{starting_concept}}")
2. Note the ontologyId and IRI
3. Get ancestors: getAncestors(ontologyId="...", classIri="...")
4. Review relationships in results:
   - directParent: Immediate parent classes
   - hierarchicalParent: Parent in asserted hierarchy
   - directAncestor: All ancestor classes

Workflow for navigating DOWN (to more specific concepts):
1. Find the starting class: searchClasses(query="{{starting_concept}}")
2. Note the ontologyId and IRI
3. Get descendants: getDescendants(ontologyId="...", classIri="...")

Understanding hierarchy types:
- "directParent" = immediate is-a relationship
- "hierarchicalParent" = structural hierarchy (may include part-of)
- "ancestors" = all classes above (transitive)
- "descendants" = all classes below (transitive)

Use pagination for large hierarchies (e.g., upper-level classes may have hundreds of descendants)
```

**Variables:** `starting_concept` (string), `direction` (enum: "ancestors" | "descendants")

---

## Priority 2: Add Resource Providers

Resources provide documentation and context that agents can access. [MCP Resources Documentation](https://modelcontextprotocol.io/docs/concepts/resources/)

### Recommended Resources

#### 1. `ontology-basics-guide`
**URI:** `ols://docs/ontology-basics`
**MIME Type:** `text/markdown`

**Content:**
```markdown
# Ontology Basics for OLS

## What are Ontologies?

Ontologies are structured vocabularies that define concepts and relationships within a domain. They provide:
- Standardized terminology
- Hierarchical organization
- Relationship definitions
- Cross-references between concepts

## Key Concepts

### Classes
The fundamental units representing concepts (e.g., "heart disease", "apoptosis", "aspirin")

### Hierarchies
Classes are organized in parent-child relationships:
- Parent (superclass): More general concept
- Child (subclass): More specific concept
- Ancestor: Any class above in the hierarchy
- Descendant: Any class below in the hierarchy

### Identifiers

**IRI (Internationalized Resource Identifier):**
- Full unique identifier
- Example: http://purl.obolibrary.org/obo/MONDO_0005015

**CURIE (Compact URI):**
- Shortened form using prefix
- Example: MONDO:0005015
- Prefix indicates source ontology

## Common Use Cases

1. **Data Annotation:** Tagging datasets with standardized terms
2. **Data Integration:** Mapping terms across different data sources
3. **Semantic Search:** Finding related concepts through hierarchy
4. **Knowledge Graphs:** Building connected domain knowledge
```

---

#### 2. `ontology-catalog`
**URI:** `ols://docs/ontology-catalog`
**MIME Type:** `application/json`

**Content:**
```json
{
  "ontologies": [
    {
      "id": "mondo",
      "name": "Mondo Disease Ontology",
      "domain": "diseases, disorders, medical conditions",
      "useCases": [
        "Disease classification",
        "Medical coding",
        "Clinical data annotation"
      ],
      "exampleTerms": ["diabetes mellitus", "heart disease", "cancer"]
    },
    {
      "id": "go",
      "name": "Gene Ontology",
      "domain": "biological processes, molecular functions, cellular components",
      "useCases": [
        "Gene annotation",
        "Functional genomics",
        "Systems biology"
      ],
      "exampleTerms": ["apoptosis", "DNA binding", "mitochondrion"]
    },
    {
      "id": "efo",
      "name": "Experimental Factor Ontology",
      "domain": "experimental variables, measurement types, disease contexts",
      "useCases": [
        "Experimental design",
        "Data annotation",
        "Clinical trial metadata"
      ],
      "exampleTerms": ["RNA sequencing", "disease staging", "age"]
    },
    {
      "id": "chebi",
      "name": "Chemical Entities of Biological Interest",
      "domain": "chemical compounds, drugs, metabolites",
      "useCases": [
        "Drug annotation",
        "Metabolomics",
        "Chemical biology"
      ],
      "exampleTerms": ["aspirin", "glucose", "ATP"]
    },
    {
      "id": "hp",
      "name": "Human Phenotype Ontology",
      "domain": "phenotypic abnormalities, clinical features",
      "useCases": [
        "Clinical diagnosis",
        "Rare disease description",
        "Patient phenotyping"
      ],
      "exampleTerms": ["seizures", "abnormal heart rate", "intellectual disability"]
    },
    {
      "id": "uberon",
      "name": "Uberon Anatomy Ontology",
      "domain": "anatomical structures across species",
      "useCases": [
        "Comparative anatomy",
        "Developmental biology",
        "Cross-species studies"
      ],
      "exampleTerms": ["heart", "brain", "liver"]
    }
  ],
  "selectionGuidance": {
    "forDiseases": ["mondo", "doid"],
    "forGenes": ["go"],
    "forChemicals": ["chebi"],
    "forPhenotypes": ["hp", "mp"],
    "forAnatomy": ["uberon", "fma"],
    "forExperiments": ["efo", "obi"]
  }
}
```

---

#### 3. `api-examples`
**URI:** `ols://docs/api-examples`
**MIME Type:** `text/markdown`

**Content:**
```markdown
# OLS MCP API Examples

## Example 1: Basic Search Workflow

**Goal:** Find information about "diabetes"

```
Step 1: Search for the term
→ searchClasses(query="diabetes", pageSize=10)

Step 2: Review results
← Returns classes from multiple ontologies (MONDO, EFO, etc.)
← Identify most relevant: MONDO:0005015 "diabetes mellitus"

Step 3: Get hierarchical context
→ getAncestors(ontologyId="mondo", classIri="http://purl.obolibrary.org/obo/MONDO_0005015")
← Shows this is a "metabolic disease" → "disease"

Step 4: Find subtypes
→ getDescendants(ontologyId="mondo", classIri="http://purl.obolibrary.org/obo/MONDO_0005015")
← Shows "type 1 diabetes", "type 2 diabetes", etc.
```

## Example 2: Cross-Ontology Search

**Goal:** Compare how different ontologies represent "inflammation"

```
Step 1: Search without ontology filter
→ searchClasses(query="inflammation")

Step 2: Group by ontology
← GO: "inflammatory response" (biological process)
← HP: "inflammation" (phenotype/symptom)
← MONDO: "inflammatory disease" (disease category)

Step 3: Understand different perspectives
- GO focuses on the biological mechanism
- HP focuses on the clinical observation
- MONDO focuses on disease classification
```

## Example 3: OpenAI Workflow

**Goal:** Quick search and detailed fetch

```
Step 1: Quick search
→ search(query="cancer")
← Returns: [
  {id: "mondo+http://...", title: "MONDO:0004992 cancer", url: "..."},
  {id: "efo+http://...", title: "EFO:0000311 cancer", url: "..."}
]

Step 2: Fetch full details
→ fetch(id="mondo+http://purl.obolibrary.org/obo/MONDO_0004992")
← Returns full entity with:
  - Complete definition
  - Hierarchy relationships
  - All metadata
```

## Example 4: Handling Pagination

**Goal:** Get all descendants of a high-level class

```
Step 1: Initial query
→ getDescendants(ontologyId="go", classIri="http://...", pageSize=100, pageNum=0)
← Returns: {items: [...], totalPages: 15, pageNum: 0}

Step 2: Iterate through pages
→ Loop pageNum from 1 to 14
→ getDescendants(..., pageNum=1)
→ getDescendants(..., pageNum=2)
...

Alternative: Request larger page size (up to reasonable limit)
→ getDescendants(..., pageSize=500)
```

## Common Patterns

### Pattern: "Find and Explore"
1. searchClasses() - Find the term
2. getAncestors() - Understand broader context
3. getDescendants() - Find related specific terms

### Pattern: "Domain Discovery"
1. listOntologies() - See what's available
2. searchClasses(ontologyId="...") - Search within domain
3. Navigate hierarchy as needed

### Pattern: "Quick Lookup"
1. search() - Fast search with compact results
2. fetch() - Get details for interesting items
```

---

#### 4. `hierarchy-types-explained`
**URI:** `ols://docs/hierarchy-types`
**MIME Type:** `text/markdown`

**Content:**
```markdown
# Understanding OLS Hierarchy Relationships

## Overview

OLS exposes multiple types of hierarchical relationships. Understanding these helps you navigate ontologies effectively.

## Relationship Types in McpClass

### directParent
**Definition:** Immediate parent class in the is-a hierarchy

**Example:**
- Class: "type 2 diabetes mellitus"
- directParent: "diabetes mellitus"

**Use when:** You want only the immediate superclass

---

### directAncestor
**Definition:** All ancestor classes (transitive closure of is-a relationships)

**Example:**
- Class: "type 2 diabetes mellitus"
- directAncestor: ["diabetes mellitus", "metabolic disease", "disease", "disease or disorder"]

**Use when:** You want the complete path to the root

---

### hierarchicalParent
**Definition:** Parent in the asserted hierarchy (may include relationships beyond is-a)

**Example:**
- Class: "heart"
- hierarchicalParent: May include "cardiovascular system" (part-of relationship)

**Use when:** You want the structural organization including compositional relationships

---

## Tools and Relationships

### getAncestors()
Returns all classes above the target in the hierarchy

**Includes:** All transitive superclasses
**Direction:** From specific → general
**Example:** "type 2 diabetes" → "diabetes" → "metabolic disease" → "disease"

### getDescendants()
Returns all classes below the target in the hierarchy

**Includes:** All transitive subclasses
**Direction:** From general → specific
**Example:** "diabetes" → "type 1 diabetes", "type 2 diabetes", "gestational diabetes"...

## Best Practices

1. **Use directParent** when you need immediate classification
2. **Use directAncestor** when you need full lineage
3. **Use hierarchicalParent** when structure matters (e.g., anatomy: organ → system → body)
4. **Use getAncestors()** to understand "what is this a type of?"
5. **Use getDescendants()** to answer "what are the types of this?"

## Gotchas

- Hierarchies can be deep (>10 levels)
- Classes can have multiple parents (multiple inheritance)
- Some classes are "obsolete" (deprecated terms)
- Cross-references may point to equivalent terms in other ontologies
```

---

## Priority 3: Enhanced Tool Descriptions

Improve tool descriptions with domain context, usage tips, and examples.

### Before and After Examples

#### Tool: searchClasses

**Before (current):**
```java
@Tool(description = "Search all classes in OLS for a query string")
```

**After (recommended):**
```java
@Tool(description = """
    Search all classes in OLS for a query string.

    Classes represent concepts in ontologies such as diseases (MONDO),
    biological processes (GO), chemical compounds (CHEBI), and more.

    Usage tips:
    - Use general terms for initial searches (e.g., "cancer", "metabolism")
    - Specify ontologyId to restrict search to a domain (e.g., mondo, go, efo)
    - Results include hierarchical relationships for context
    - Default page size is 20; increase for comprehensive results

    Common ontology IDs:
    - mondo: Diseases and medical conditions
    - go: Biological processes, molecular functions, cellular components
    - chebi: Chemical entities and drugs
    - efo: Experimental factors and phenotypes
    - hp: Human phenotypes and symptoms

    Example: searchClasses(query="diabetes", ontologyId="mondo", pageSize=10)
    """)
```

---

#### Tool: getAncestors

**Before (current):**
```java
@Tool(description = "Get all ancestors for a class in OLS")
```

**After (recommended):**
```java
@Tool(description = """
    Get all ancestors (parent classes) for a class in OLS.

    Ancestors represent more general concepts in the ontology hierarchy.
    This tool performs transitive closure, returning all classes above
    the target class, not just immediate parents.

    Use this to:
    - Understand broader categories and classifications
    - Find the root concept of a hierarchy
    - Discover related high-level terms

    Results include:
    - directParent: Immediate parent classes
    - directAncestor: All ancestor classes up the tree
    - hierarchicalParent: Structural hierarchy (may include part-of relationships)

    Note: The class IRI must be exact and properly formatted. Obtain IRIs
    from searchClasses results.

    Example: getAncestors(
        ontologyId="mondo",
        classIri="http://purl.obolibrary.org/obo/MONDO_0005015",
        pageSize=50
    )

    For "type 2 diabetes", this might return:
    - diabetes mellitus
    - metabolic disease
    - disease
    - disease or disorder
    """)
```

---

#### Parameter: ontologyId

**Before (current):**
```java
@ToolParam(required=false) String ontologyId
```

**After (recommended):**
```java
@ToolParam(
    required=false,
    description = """
        The ontology identifier to restrict search scope.

        Leave empty to search across all ontologies.

        Popular ontologies:
        - mondo: Diseases
        - go: Gene function and biology
        - chebi: Chemistry
        - efo: Experimental factors
        - hp: Human phenotypes
        - uberon: Anatomy

        Get full list with listOntologies() tool.
        """
) String ontologyId
```

---

## Implementation Guide

### For Spring AI MCP

#### 1. Add Prompt Provider

Create `McpPromptService.java`:

```java
package uk.ac.ebi.spot.ols.controller.mcp;

import org.springframework.ai.mcp.server.prompt.Prompt;
import org.springframework.ai.mcp.server.prompt.PromptProvider;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public class McpPromptService implements PromptProvider {

    @Override
    public List<Prompt> getPrompts() {
        return List.of(
            createExploreConceptPrompt(),
            createDiseaseLookupPrompt(),
            createCompareOntologiesPrompt(),
            createOpenAIWorkflowPrompt(),
            createNavigateTreePrompt()
        );
    }

    private Prompt createExploreConceptPrompt() {
        return Prompt.builder()
            .name("explore_concept_hierarchy")
            .description("Guide for exploring a concept and its hierarchical relationships")
            .arguments(Map.of("concept", "The concept to explore"))
            .template("""
                You are helping explore the concept: {{concept}}

                [Full template content here...]
                """)
            .build();
    }

    // Additional prompt creation methods...
}
```

Register in `Ols4Backend.java`:
```java
@Bean
public PromptProvider mcpPrompts(McpPromptService service) {
    return service;
}
```

---

#### 2. Add Resource Provider

Create `McpResourceService.java`:

```java
package uk.ac.ebi.spot.ols.controller.mcp;

import org.springframework.ai.mcp.server.resource.Resource;
import org.springframework.ai.mcp.server.resource.ResourceProvider;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class McpResourceService implements ResourceProvider {

    @Override
    public List<Resource> getResources() {
        return List.of(
            createOntologyBasicsResource(),
            createCatalogResource(),
            createExamplesResource(),
            createHierarchyTypesResource()
        );
    }

    private Resource createOntologyBasicsResource() {
        return Resource.builder()
            .uri("ols://docs/ontology-basics")
            .name("Ontology Basics Guide")
            .description("Introduction to ontology concepts for OLS")
            .mimeType("text/markdown")
            .content(loadOntologyBasicsContent())
            .build();
    }

    private String loadOntologyBasicsContent() {
        // Load from file or return inline content
        return """
            # Ontology Basics for OLS
            [Full content here...]
            """;
    }

    // Additional resource creation methods...
}
```

Register in `Ols4Backend.java`:
```java
@Bean
public ResourceProvider mcpResources(McpResourceService service) {
    return service;
}
```

---

#### 3. Enhanced Tool Descriptions

Update existing service files:

**McpClassService.java:**
```java
@Tool(description = """
    Search all classes in OLS for a query string.

    [Enhanced description with context, tips, examples...]
    """)
McpPage<McpClass> searchClasses(
    @ToolParam(description = "Search query string (e.g., 'diabetes', 'apoptosis')")
    String query,

    @ToolParam(
        required=false,
        description = "Ontology ID to restrict search (e.g., 'mondo', 'go'). Leave empty to search all ontologies."
    )
    String ontologyId,

    @ToolParam(
        required=false,
        description = "Page number for pagination (0-indexed, default: 0)"
    )
    Integer pageNum,

    @ToolParam(
        required=false,
        description = "Results per page (default: 20, recommended: 10-100)"
    )
    Integer pageSize,

    @ToolParam(
        required=false,
        description = "Language code for labels/definitions (default: 'en')"
    )
    String lang
)
```

---

## Priority 4: Add Frontend Documentation

Update `frontend/src/pages/MCP.tsx` to include agent usage guidance:

```typescript
<section>
  <h2>For AI Agent Developers</h2>

  <h3>Available Prompts</h3>
  <p>The MCP server provides predefined prompt templates to guide common workflows:</p>
  <ul>
    <li><code>explore_concept_hierarchy</code> - Navigate term relationships</li>
    <li><code>find_disease_hierarchy</code> - Disease-specific searches</li>
    <li><code>compare_terms_across_ontologies</code> - Cross-ontology comparison</li>
  </ul>

  <h3>Available Resources</h3>
  <p>Documentation accessible to agents:</p>
  <ul>
    <li><code>ols://docs/ontology-basics</code> - Ontology fundamentals</li>
    <li><code>ols://docs/ontology-catalog</code> - Available ontologies guide</li>
    <li><code>ols://docs/api-examples</code> - Common usage patterns</li>
  </ul>

  <h3>Quick Start for Agents</h3>
  <pre>{`
1. Access resource: ols://docs/ontology-basics
2. List ontologies: listOntologies()
3. Search for terms: searchClasses(query="your term")
4. Navigate hierarchy: getAncestors() or getDescendants()
  `}</pre>
</section>
```

---

## Testing Recommendations

### 1. Test with Claude Desktop
Configure Claude Desktop to connect to the MCP server and verify:
- Prompts are visible and usable
- Resources are accessible
- Tool descriptions are clear
- Workflows function as expected

### 2. Test with Different Agent Types
- OpenAI agents (using search/fetch pattern)
- Claude agents (using full tool set)
- Generic MCP clients

### 3. Measure Effectiveness
Compare agent performance with and without prompts/resources:
- Task completion rate
- Query efficiency (fewer unnecessary calls)
- Result quality
- Handling of edge cases

---

## Expected Benefits

### For Agents
1. **Better understanding** of ontology domain concepts
2. **More efficient workflows** following best practices
3. **Improved parameter selection** based on use case
4. **Reduced errors** from misunderstanding tools
5. **Self-service learning** through resources

### For Users
1. **More reliable results** from agent queries
2. **Better explanations** when agents cite OLS data
3. **Faster task completion** with guided workflows
4. **Reduced need for human intervention** and clarification

### For OLS
1. **Increased adoption** by AI agent developers
2. **Standardized usage patterns** across agents
3. **Better API utilization** (fewer wasteful queries)
4. **Competitive advantage** over MCP servers without context

---

## Maintenance Notes

1. **Version prompts** - Include version numbers for tracking changes
2. **Update resources** when ontologies change or new ones are added
3. **Monitor agent logs** to identify confusion points
4. **Iterate on descriptions** based on actual usage patterns
5. **Keep examples current** with latest API capabilities

---

## References

- [MCP Prompts Specification](https://modelcontextprotocol.io/specification/2025-06-18/server/prompts)
- [MCP Resources Documentation](https://modelcontextprotocol.info/docs/concepts/prompts/)
- [MCP Best Practices Guide](https://composio.dev/blog/how-to-effectively-use-prompts-resources-and-tools-in-mcp)
- [MCP Features Guide](https://workos.com/blog/mcp-features-guide)
- [OpenAI MCP Documentation](https://openai.github.io/openai-agents-python/mcp/)
