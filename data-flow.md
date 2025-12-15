# TDP Platform - Data Flow Documentation

## Overview

This document describes the data flow patterns for key operations in the TDP platform, including graph queries, analytics queries, GSEA analysis, and data commons access.

---

## 1. Graph Query Flow (Gene Interaction Network)

User queries for protein-protein interaction networks with configurable interaction types (PPI, FUN_PPI, BIO_GRID, INT_ACT) and optional algorithm processing.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant F as Frontend
    participant G as API Gateway
    participant B as Backend (NestJS)
    participant R as Redis Cache
    participant N as Neo4j

    U->>F: Enter genes, select interaction types
    F->>G: GraphQL: getGeneInteractions
    G->>B: Forward to /graphql
    
    B->>B: Generate graph hash (SHA256)
    B->>B: Extract/create user-id cookie
    
    B->>N: Check if graph exists (gds.graph.exists)
    
    alt Graph Exists in GDS
        N-->>B: exists = true
        B->>N: Query cached graph projection
    else Graph Not Exists
        N-->>B: exists = false
        B->>N: MATCH genes + relationships
        B->>N: Create GDS graph projection
    end
    
    N-->>B: Raw nodes and edges
    
    B->>B: Merge edges, calculate avg scores
    B->>R: Bind graph to user session
    R->>R: Set TTL (REDIS_KEY_EXPIRY)
    
    B-->>G: GraphQL response
    G-->>F: JSON {genes, links, graphName}
    F->>F: Build Graphology graph
    F->>F: Render with Sigma.js
    F-->>U: Interactive network visualization
```

### Key Implementation Details

**File References**:
- Query Definition: `frontend/lib/gql.ts` → `GENE_GRAPH_QUERY`
- Resolver: `backend/src/gql/gql.resolver.ts` → `getGeneInteractions()`
- Service: `backend/src/gql/gql.service.ts` → `getGeneInteractions()`
- Cypher Queries: `backend/src/neo4j/neo4j.constants.ts`

**Graph Hash Computation**:
```typescript
// backend/src/gql/gql.service.ts
computeHash(query: string) {
  return createHash('sha256').update(query).digest('hex');
}
```

**Interaction Types**:
| Type | Description |
|------|-------------|
| `PPI` | Protein-Protein Interaction |
| `FUN_PPI` | Functional PPI |
| `BIO_GRID` | BioGRID interactions |
| `INT_ACT` | IntAct interactions |

**Order Modes**:
- `0`: Direct connections only (genes in input list)
- `1`: First-order neighbors (expand one hop)
- `2`: Second-order (internally converts to order 0 with expanded list)

---

## 2. Analytics Query Flow (Gene Properties & Association Scores)

Fetching gene properties, association scores, and differential expression data from ClickHouse.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant F as Frontend
    participant G as API Gateway
    participant B as Backend (NestJS)
    participant CH as ClickHouse

    U->>F: Select genes, disease, properties
    F->>G: GraphQL: geneProperties
    G->>B: Forward to /graphql
    
    B->>B: Parse DataRequired config
    
    loop For each property category
        B->>CH: SQL query with gene_ids filter
        Note over CH: SELECT from category table<br/>WHERE gene_id IN (batch)
        CH-->>B: Result rows
    end
    
    B->>B: Aggregate by gene ID
    B-->>G: GraphQL response
    G-->>F: JSON {geneProperties}
    F->>F: Render heatmap/table
    F-->>U: Gene property visualization
```

### Property Categories

| Category | Table | Disease-Dependent |
|----------|-------|-------------------|
| `OPEN_TARGETS` | `overall_association_score` | Yes |
| `DIFFERENTIAL_EXPRESSION` | `differential_expression` | Yes |
| `GENETICS` | `genetics` | Yes |
| `DRUGGABILITY` | `druggability` | No |
| `PATHWAY` | `pathway` | No |
| `TISSUE_EXPRESSION` | `tissue_specificity` | No |
| `OT_PRIORITIZATION` | `target_prioritization_factors` | No |

### Target-Disease Association Table

```mermaid
sequenceDiagram
    autonumber
    participant F as Frontend
    participant B as Backend
    participant CH as ClickHouse

    F->>B: GraphQL: targetDiseaseAssociationTable
    B->>CH: Query materialized view
    Note over CH: SELECT FROM<br/>mv_datasource_association_score_overall_association_score<br/>WHERE disease_id = X AND gene_id IN (...)
    CH-->>B: Joined scores (overall + per-datasource)
    B->>B: Parse datasource scores
    B-->>F: Paginated association table
```

**File References**:
- Resolver: `backend/src/gql/clickhouse.resolver.ts`
- Service: `backend/src/clickhouse/clickhouse.service.ts`
- Migrations: `backend/src/clickhouse/migrations/`

---

## 3. GSEA Analysis Flow

Gene Set Enrichment Analysis using KEGG and Reactome pathway databases.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant F as Frontend
    participant G as API Gateway
    participant GS as GSEA Service (Python)
    participant GMT as GMT Files

    U->>F: Select genes for enrichment
    F->>F: Prepare gene list
    F->>G: POST /api/gsea/ {gene_list}
    G->>GS: Forward request
    
    GS->>GMT: Load pathway definitions
    Note over GMT: pathway_kegg_gsea.gmt<br/>pathway_reactome_gsea.gmt
    
    GS->>GS: Run gseapy.enrich()
    GS->>GS: Sort by P-value
    GS->>GS: Format scores (scientific notation)
    
    GS-->>G: JSON [{Pathway, P-value, Genes, ...}]
    G-->>F: Forward response
    F->>F: Render enrichment table
    F-->>U: GSEA results with pathway details
```

### Response Format

```json
[
  {
    "Pathway": "KEGG_OXIDATIVE_PHOSPHORYLATION",
    "Overlap": "15/120",
    "P-value": "1.23e-05",
    "Adjusted P-value": "4.56e-04",
    "Odds Ratio": "3.45",
    "Combined Score": "45.67",
    "Genes": "ATP5A1,ATP5B,COX5A,..."
  }
]
```

**File References**:
- Service: `gsea/app.py`
- Pathway Files: `gsea/pathway_kegg_gsea.gmt`, `gsea/pathway_reactome_gsea.gmt`

---

## 4. Data Commons Access Flow

Authenticated access to project-specific data files (expression matrices, sample sheets, differential expression).

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant F as Frontend
    participant G as API Gateway
    participant B as Backend (NestJS)
    participant PG as PostgreSQL
    participant FS as File System

    U->>F: Navigate to Data Commons
    F->>G: GET /api/nestjs/data-commons/structure
    G->>B: Forward request
    B->>FS: Scan /app/src/data-commons/data
    FS-->>B: Directory tree (groups/programs/projects)
    B-->>F: JSON structure

    U->>F: Select protected project
    F->>G: GET /api/nestjs/data-commons/project/.../verify-auth
    G->>B: Forward with cookies
    
    B->>B: Verify JWT from cookie
    alt Valid JWT Session
        B->>PG: Check Combination exists
        PG-->>B: Session valid
        B-->>F: 200 OK
    else No/Invalid JWT
        B-->>F: 401 Unauthorized
        F->>U: Show password prompt
        U->>F: Enter password
        F->>G: POST /api/nestjs/data-commons/project/.../password
        G->>B: {password}
        B->>PG: Validate combination
        alt Valid Password
            PG-->>B: Match found
            B->>PG: Create Session + Combination
            B->>B: Generate JWT
            B-->>F: Set-Cookie: session-token
        else Invalid
            B-->>F: 401 Unauthorized
        end
    end

    F->>G: GET /api/nestjs/data-commons/project/.../files/{filename}
    G->>B: Forward with cookie
    B->>B: Verify JWT
    B->>FS: Read file
    FS-->>B: File content
    B-->>F: Stream response
    F-->>U: Display/download file
```

### Directory Structure

```
data-commons/data/
├── {group}/
│   ├── {program}/
│   │   ├── {project}/
│   │   │   ├── description.md
│   │   │   ├── gene_counts.csv
│   │   │   ├── transcript_counts.csv
│   │   │   ├── sample_sheet.csv
│   │   │   └── de_files/
│   │   │       ├── differential_expression_contrast1.csv
│   │   │       └── differential_expression_contrast2.csv
```

### Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/data-commons/structure` | List all groups/programs/projects |
| `GET` | `/data-commons/project/:g/:p/:proj/description` | Project README |
| `GET` | `/data-commons/project/:g/:p/:proj/files/:file` | Download file |
| `GET` | `/data-commons/project/:g/:p/:proj/deFile/:file` | Get DE file |
| `GET` | `/data-commons/project/:g/:p/:proj/preview/:file` | Preview file (head) |
| `POST` | `/data-commons/project/:g/:p/:proj/password` | Authenticate |
| `GET` | `/data-commons/project/:g/:p/:proj/verify-auth` | Check auth status |

**File References**:
- Controller: `backend/src/data-commons/dataCommons.controller.ts`
- Service: `backend/src/data-commons/dataCommons.service.ts`
- Prisma Schema: `backend/prisma/schema.prisma`

---

## 5. Differential Expression Visualization Flow

Processing and visualizing volcano plots from DE files.

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant F as Frontend
    participant B as Backend
    participant FS as File System

    U->>F: Open DE Analysis tab
    F->>B: GET /data-commons/project/.../initializedFiles
    B->>FS: List DE files
    FS-->>B: File list
    B-->>F: {geneDeFiles, transcriptDeFiles}

    loop For each DE file
        F->>B: GET /data-commons/project/.../deFile/{filename}
        B->>FS: Read TSV/CSV
        FS-->>B: Raw content
        B-->>F: File content
    end

    F->>F: Parse with PapaParse
    F->>F: Extract contrasts from filenames
    F->>F: Apply thresholds (logFC, p-value)
    F->>F: Categorize points (Up/Down/None)
    
    F->>F: Render Plotly volcano plot
    F-->>U: Interactive volcano plots

    U->>F: Adjust threshold sliders
    F->>F: Re-categorize points
    F-->>U: Updated visualization
```

### File Naming Convention

```regex
/^(?:.*)(?:(?:differential|diff)(?:[-_ ]?(?:exp|expression))?|(?:differential|de))(?:[-_ ]?)(.+?)\.(csv|tsv|xls|xlsx|txt)$/i
```

**Examples**:
- `differential_expression_contrast1.csv` → Contrast: `contrast1`
- `diff_exp_drug_vs_control.tsv` → Contrast: `drug_vs_control`

**File References**:
- Parser: `frontend/components/data-commons/DifferentialExpression/utils.ts`
- Hooks: `frontend/components/data-commons/DifferentialExpression/hooks.ts`
- Component: `frontend/components/data-commons/DifferentialExpression/DE.tsx`

---

## 6. Redis Cache & Graph Lifecycle

### Cache Key Management

```mermaid
sequenceDiagram
    autonumber
    participant B as Backend
    participant R as Redis
    participant N as Neo4j

    Note over B,N: Graph Creation
    B->>R: INCR {graphHash}
    B->>R: EXPIRE {graphHash} 900
    B->>R: SET user:{userId} {graphHash}
    
    Note over B,N: User Session Binding
    B->>R: GET user:{userId}
    R-->>B: Current graph binding
    B->>R: DECR {oldGraphHash}
    alt Count reaches 0
        B->>R: DEL {oldGraphHash}
        B->>N: CALL gds.graph.drop(oldGraphHash)
    end

    Note over R,N: TTL Expiration (automatic)
    R->>R: Key expires
    R-->>B: Keyspace notification
    B->>N: CALL gds.graph.drop(expiredKey)
```

### Cache Configuration

```properties
REDIS_KEY_EXPIRY=900      # 15 minutes - graph cache TTL
REDIS_USER_EXPIRY=7200    # 2 hours - user session TTL
```

**File References**:
- Redis Service: `backend/src/redis/redis.service.ts`
- Neo4j Service: `backend/src/neo4j/neo4j.service.ts` → `bindGraph()`, `onKeyExpiration()`

---

## 7. Data Ingestion Pipeline

Batch data loading from CSV/TSV files into databases.

### Pipeline Flow

```mermaid
flowchart TB
    subgraph Input["Input Files"]
        CSV["CSV/TSV Files"]
        GMT["GMT Pathway Files"]
    end

    subgraph CLI["Python CLI (scripts/cli.py)"]
        Parse["Parse & Validate"]
        Transform["Transform Data"]
        Batch["Batch Insert"]
    end

    subgraph Targets["Target Databases"]
        Neo4j["Neo4j\n(Genes, Diseases, Properties)"]
        ClickHouse["ClickHouse\n(Scores, DE, Analytics)"]
    end

    subgraph PostProcess["Post-Processing"]
        MV["Refresh Materialized View"]
        Props["Seed Neo4j Properties"]
    end

    CSV --> Parse
    GMT --> Parse
    Parse --> Transform
    Transform --> Batch
    Batch --> Neo4j
    Batch --> ClickHouse
    ClickHouse --> MV
    Neo4j --> Props
```

### CLI Commands

```bash
# Connect to databases
python cli.py --help

# Seed ClickHouse tables
python cli.py seed-clickhouse \
  --table overall_association_score \
  --file data/association_scores.csv

# Seed Neo4j properties
python cli.py seed-neo4j-properties \
  --file data/property_definitions.csv

# Refresh materialized view
python cli.py refresh-mv
```

### Table Mappings

| Prefix | ClickHouse Table |
|--------|------------------|
| `Pathway` | `pathway` |
| `Druggability` | `druggability` |
| `TE` | `tissue_specificity` |
| `OT_Prioritization` | `target_prioritization_factors` |
| `Genetics` | `genetics` |
| `LogFC` | `differential_expression` |
| `OpenTargets` | `overall_association_score`, `datasource_association_score` |

**File References**:
- CLI: `scripts/cli.py`
- Migrations: `backend/src/clickhouse/migrations/`

---

## 8. Algorithm Execution Flow (Leiden Clustering)

### Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant F as Frontend
    participant B as Backend
    participant R as Redis
    participant N as Neo4j GDS

    F->>B: Request clustering (resolution param)
    B->>R: Get user's current graph
    R-->>B: graphName
    
    B->>N: Check graph exists
    N-->>B: exists = true
    
    B->>N: CALL gds.leiden.stream()
    Note over N: Run Leiden algorithm<br/>on in-memory graph projection
    N-->>B: {nodeId, communityId}[]
    
    B->>B: Map nodeIds to gene IDs
    B->>B: Generate community colors
    B-->>F: {geneId, communityId, color}[]
    F->>F: Apply colors to nodes
```

### Algorithm Parameters

```cypher
CALL gds.leiden.stream($graphName, {
  relationshipWeightProperty: "score",
  gamma: $resolution,
  minCommunitySize: 3,
  logProgress: false
})
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).ID AS ID, communityId
```

**File References**:
- Service: `backend/src/algorithm/algorithm.service.ts`
- Constants: `backend/src/neo4j/neo4j.constants.ts` → `LEIDEN_QUERY`
