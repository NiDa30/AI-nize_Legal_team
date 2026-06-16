# AI-nize Legal Team: Repository Architecture Design

## 1. Repository Tree

```text
ai-legal-advisor/
├── .github/                        # GitHub Actions (CI/CD), Issue Templates
├── docs/                           # Architecture, Runbooks, API Specs
├── knowledge_base/                 # Source of Truth for Legal Documents
│   ├── labor_law/
│   │   ├── laws/
│   │   ├── decrees/
│   │   └── circulars/
│   ├── real_estate_law/
│   │   ├── laws/
│   │   ├── decrees/
│   │   └── circulars/
│   └── accounting_law/
│       ├── laws/
│       ├── decrees/
│       └── circulars/
├── metadata/                       # Tracking & Governance
│   ├── schemas/                    # JSON Schemas for validation
│   ├── registries/                 # Master list of all documents (CSV/JSON)
│   ├── mappings/                   # Amendment & hierarchy mapping
│   └── changelogs/                 # Legal update history
├── prompts/                        # LLM Prompt Library (Gemini)
│   ├── system_prompts/             # Core agent personas
│   ├── extraction_prompts/         # RAG retrieval instructions
│   ├── drafting_prompts/           # Contract clause generation
│   └── validation_prompts/         # Hallucination & compliance checks
├── templates/                      # Standard Contract Templates
│   ├── labor/
│   ├── real_estate/
│   └── accounting/
├── rag_pipeline/                   # Python - Data Ingestion & Processing
│   ├── scrapers/                   # Fetching from vbpl.vn
│   ├── chunkers/                   # Legal-specific text chunking
│   ├── embedders/                  # Embedding generation
│   └── tests/
├── services/                       # Spring Boot AI Application
│   ├── api-gateway/                # Routing & Auth
│   ├── legal-search-agent/         # RAG Query service
│   ├── compliance-agent/           # Verification service
│   └── draft-generation-agent/     # Output generation service
├── qdrant_config/                  # Vector DB configurations
│   ├── collections/                # Collection schema definitions
│   └── migrations/                 # Scripts to update vector indexes
├── docker-compose.yml              # Local development setup
├── Makefile                        # Build automation commands
└── README.md                       # Repository overview
```

## 2. Folder Purpose Table

| Folder / Path | Purpose |
| :--- | :--- |
| `docs/` | Comprehensive technical documentation, ADRs (Architecture Decision Records), and developer onboarding guides. |
| `knowledge_base/` | Stores the raw (Markdown/TXT/PDF) verified legal documents organized by domain and legal hierarchy. |
| `metadata/` | Crucial for RAG. Stores the structured metadata (dates, status, relationships) defining the legal validity of the `knowledge_base`. |
| `prompts/` | Version-controlled prompt library. Essential for tuning Gemini models and standardizing agent behavior. |
| `templates/` | Bilingual contract templates (VN/EN) used as baseline structures by the `draft-generation-agent`. |
| `rag_pipeline/` | The data engineering layer. Responsible for transforming raw legal text into vector embeddings and updating Qdrant. |
| `services/` | The backend application layer (Spring Boot). Hosts the business logic, AI orchestration (Spring AI/LangChain4j), and REST APIs. |
| `qdrant_config/` | Infrastructure-as-Code for the Qdrant Vector Database, defining payloads, indexing parameters, and collections. |

## 3. Naming Conventions

*   **Directories:** `snake_case` (e.g., `real_estate_law`, `system_prompts`).
*   **Spring Boot Services:** `kebab-case` (e.g., `legal-search-agent`).
*   **Legal Documents:** `[DocumentType]_[Number]_[Year]_[Issuer].md` (e.g., `Law_45_2019_QH14.md`, `Decree_145_2020_ND-CP.md`).
*   **Metadata Files:** Match the legal document name exactly, but use `.json` (e.g., `Law_45_2019_QH14.meta.json`).
*   **Prompts:** `[Action]_[Domain]_[Version].txt` (e.g., `draft_labor_clause_v1.txt`).

## 4. Metadata Schema (JSON)

Every document in the `knowledge_base` has an associated metadata file to enable precise filtering in Qdrant and verification by the AI.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Legal Document Metadata",
  "type": "object",
  "properties": {
    "document_id": { "type": "string", "description": "Official number (e.g., 45/2019/QH14)" },
    "domain": { "type": "string", "enum": ["Labor", "Real Estate", "Accounting"] },
    "type": { "type": "string", "enum": ["Law", "Decree", "Circular", "Resolution"] },
    "issuer": { "type": "string", "description": "Issuing authority" },
    "issue_date": { "type": "string", "format": "date" },
    "effective_date": { "type": "string", "format": "date" },
    "expiration_date": { "type": ["string", "null"], "format": "date" },
    "status": { "type": "string", "enum": ["Active", "Expired", "Partially Expired", "Future"] },
    "official_url": { "type": "string", "format": "uri" },
    "tags": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["document_id", "domain", "type", "effective_date", "status"]
}
```
*Example implementation: `metadata/schemas/document_schema.json`*

## 5. Legal Document Registry Schema

A central manifest tracking all documents to ensure complete coverage and identify missing amendments. Recommended format: `JSON Lines` or `CSV`.

```json
[
  {
    "id": "LAW-LABOR-2019",
    "document_id": "45/2019/QH14",
    "name_vn": "Bộ luật Lao động 2019",
    "status": "Active",
    "last_verified": "2026-06-01T10:00:00Z",
    "checksum": "a1b2c3d4..."
  },
  {
    "id": "DECREE-LABOR-WAGE-2020",
    "document_id": "145/2020/NĐ-CP",
    "name_vn": "Nghị định 145/2020/NĐ-CP quy định chi tiết Bộ luật Lao động",
    "status": "Active",
    "last_verified": "2026-06-01T10:05:00Z",
    "checksum": "e5f6g7h8..."
  }
]
```
*Example implementation: `metadata/registries/labor_registry.jsonl`*

## 6. Regulation Mapping Schema

Crucial for the "Check Amendments" workflow. Maps relationships between base laws and their modifying decrees/circulars.

```json
{
  "base_document": "45/2019/QH14",
  "amended_by": [],
  "guided_by": [
    {
      "document_id": "145/2020/NĐ-CP",
      "scope": "Working conditions, labor relations",
      "specific_articles_guided": ["Article 90", "Article 91"]
    },
    {
      "document_id": "152/2020/NĐ-CP",
      "scope": "Foreign workers in Vietnam",
      "specific_articles_guided": ["Article 152"]
    }
  ],
  "supersedes": ["10/2012/QH13"]
}
```
*Example implementation: `metadata/mappings/map_45_2019_QH14.json`*

## 7. Changelog Structure

Tracks when a legal document's status changes in the system (e.g., a new decree is published).

```markdown
# Legal Database Changelog

## [2026-06-15] - Real Estate Domain Update
### Added
- `Decree_XX_2026_ND-CP.md`: New decree guiding the Land Law 2024.
### Changed
- `metadata/registries/real_estate_registry.jsonl`: Status of `Decree_43_2014_ND-CP` changed to "Expired".
### Deprecated
- `Decree_43_2014_ND-CP.md` moved to archive layer in Qdrant.
```
*Example implementation: `metadata/changelogs/CHANGELOG.md`*

## 8. Prompt Library Structure

Prompts are treated as code and versioned.

```text
prompts/
├── system_prompts/
│   ├── Legal_Advisor_Gemini_v1.txt    (Persona definition, tone, constraints)
│   └── Compliance_Auditor_Gemini_v1.txt (Verification persona)
├── extraction_prompts/
│   └── Qdrant_Query_Formulator_v2.txt (Translates user intent into vector search params)
├── drafting_prompts/
│   ├── Labor_Clause_Bilingual_v1.txt
│   └── Real_Estate_Contract_v1.txt
└── validation_prompts/
    └── Hallucination_Check_v1.txt     (Checks if draft strictly entails source text)
```

**Prompt Example (`Drafting_Prompts/Labor_Clause_Bilingual_v1.txt`):**
> You are a Vietnamese Legal Expert. Draft a contract clause based ONLY on the provided <context>. 
> Constraints:
> 1. Use formal legal English and Vietnamese.
> 2. Cite the exact Article and Document ID.
> 3. Do not invent terms. If the context does not cover the request, state: "Insufficient legal basis."
> 
> <context>
> {{RETRIEVED_QDRANT_CHUNKS}}
> </context>
> <user_request>
> {{USER_QUERY}}
> </user_request>

## 9. Template Library Structure

```text
templates/
├── labor/
│   ├── Standard_Employment_Contract.json
│   └── NDA_Clause.json
```
Templates contain variable placeholders (e.g., `{{SALARY}}`, `{{PROBATION_PERIOD}}`) that the `draft-generation-agent` targets.

## 10. Future Qdrant Integration Structure

Qdrant requires strict payload schema definitions for hybrid search (vector + keyword/filter).

```text
qdrant_config/
├── collections/
│   ├── labor_law_collection.json
│   ├── real_estate_law_collection.json
│   └── accounting_law_collection.json
└── payload_indexes/
    └── create_indexes.sh  (Creates keyword indexes on 'status', 'document_id', 'effective_date')
```

**Qdrant Collection Schema (`labor_law_collection.json`):**
Defines payload requirements:
- `vector_size`: 3072 (if using `text-embedding-3-large`)
- `payload_schema`: 
  - `document_id` (Keyword)
  - `article_number` (Keyword)
  - `effective_date` (Datetime)
  - `status` (Keyword - Must be 'Active')

## 11. Future AI Agent Integration Structure (Spring Boot)

Using **Spring AI** or **LangChain4j**, the services architecture orchestrates the flow.

```text
services/
├── compliance-agent/
│   ├── src/main/java/com/ainize/legal/compliance/
│   │   ├── agent/                 # Core Agent logic (LangChain4j Agent implementations)
│   │   ├── tools/                 # Functions agents can call (e.g., checkDate(), verifyCitation())
│   │   ├── model/                 # DTOs
│   │   └── config/                # Gemini API Client config
│   └── pom.xml
├── legal-search-agent/
│   ├── src/main/java/com/ainize/legal/search/
│   │   ├── qdrant/                # Qdrant client & Hybrid Search logic
│   │   ├── query/                 # Query decomposition & routing
│   │   └── llm/                   # Gemini integration for synthesis
│   └── pom.xml
```

## 12. Best Practices & Future Expansion Strategy

1. **Immutable Context:** Once a legal document is ingested into `knowledge_base`, it is immutable. Amendments create *new* documents and update the `mappings/` schemas.
2. **Metadata-Driven Retrieval (Pre-filtering):** Before vector similarity search occurs, the `legal-search-agent` MUST apply Qdrant payload filters to exclude any document where `status != "Active"` or `effective_date > current_date`. This guarantees 100% compliance.
3. **Gemini Gems Support:** The `prompts/system_prompts/` and `docs/` folders are designed to be easily exportable to configure customized Gemini Gems for non-technical stakeholders (e.g., HR managers using a Gem directly).
4. **CI/CD for Knowledge:** Any PR updating the `knowledge_base/` must trigger a GitHub Action that:
   - Validates the JSON metadata schema.
   - Triggers the `rag_pipeline` to chunk and embed the new text.
   - Pushes the new vectors to a staging Qdrant cluster.
5. **Future Expansion:** Adding a new domain (e.g., "Corporate Law") only requires adding a new folder under `knowledge_base`, registering it in `registries`, and configuring a new Qdrant collection. The Spring Boot Agents dynamically route based on the `domain` metadata.
