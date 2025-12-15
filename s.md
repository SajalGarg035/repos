# Target Discovery Platform (TDP) - System Context & Architecture

## 1. System Overview
This document outlines the high-level architecture and system context for the Target Discovery Platform (TDP). It details the interactions between the end-user (Researcher), the web browser, the internal microservices, and the distinct data persistence layers.

## 2. Architecture Diagram (C4 Context Model)

```mermaid
%% C4 System Context Diagram for TDP
%% Style Definitions for Read vs Write distinction
graph TD
    %% --- Style Classes ---
    classDef readFlow stroke:#0277bd,stroke-width:2px,color:#0277bd,fill:none;
    classDef writeFlow stroke:#e65100,stroke-width:2px,color:#e65100,fill:none;
    classDef person fill:#08427b,stroke:#052e56,color:white;
    classDef externalSystem fill:#999999,stroke:#666666,color:white;
    classDef internalComponent fill:#1168bd,stroke:#0b4884,color:white;
    classDef internalDatabase fill:#2f95d7,stroke:#0b4884,color:white,shape:cyl;

    %% --- Actors and External Systems ---
    User["👤 End User<br/>(Researcher/Scientist)"]:::person
    Browser["🌐 User's Web Browser<br/>(Entry Point)"]:::externalSystem

    %% --- TDP System Boundary ---
    subgraph tdp_boundary["🏢 Target Discovery Platform (TDP) System Boundary"]
        direction TB
        
        %% Entry Point Layer
        Nginx["🛡️ API Gateway<br/>(Nginx)<br/>Routes requests"]:::internalComponent

        %% Service Layer
        Frontend["💻 Frontend Service<br/>(Next.js)<br/>Serves UI, handles interactions"]:::internalComponent
        NestJS["⚙️ Backend Service<br/>(NestJS)<br/>Business logic, Auth, GraphQL API"]:::internalComponent
        GSEA["🔬 GSEA Service<br/>(Python/FastAPI)<br/>Performs enrichment analysis"]:::internalComponent

        %% Data Store Layer
        subgraph DataStores["💾 Data Stores"]
            Redis[("⚡ Redis<br/>(Cache Layer)<br/>Caches frequently accessed results")]:::internalDatabase
            Neo4j[("🕸️ Neo4j<br/>(Graph DB)<br/>Genes, diseases, interactions")]:::internalDatabase
            ClickHouse[("📊 ClickHouse<br/>(Analytics DB)<br/>Gene properties, associations")]:::internalDatabase
            Postgres[("🗄️ PostgreSQL<br/>(Relational DB)<br/>Sessions, feedback storage")]:::internalDatabase
        end

        %% Internal Processing Notes
        noteGSEA[/"Internal: Uses gseapy & GMT pathway files"/]
        GSEA -.-> noteGSEA
    end

    %% --- Flows outside boundary ---
    User ==>"HTTPS"==> Browser

    %% --- Flows crossing boundary ---
    Browser ==>"HTTPS (Port 443/80)"==> Nginx
    linkStyle 1 stroke:#e65100,stroke-width:3px; %% Initial connection is mixed

    %% --- Internal System Flows ---

    %% 1. API Gateway Routing
    Nginx --"Routes UI requests"--> Frontend
    Nginx --"Routes /api/nestjs/*"--> NestJS
    Nginx --"Routes /api/gsea/*"--> GSEA

    %% 2. Frontend Interactions (Mixed Read/Write)
    Frontend --"GraphQL Queries (Genes, Interactions)<br/>[Read]"--> NestJS:::readFlow
    Frontend --"REST (Data Commons access)<br/>[Read]"--> NestJS:::readFlow
    Frontend --"POST Gene Lists<br/>[Write]"--> GSEA:::writeFlow
    Frontend --"Submit Feedback<br/>[Write]"--> NestJS:::writeFlow

    %% 3. Backend to Data Stores (Read Flows - Blue)
    NestJS --"Bolt (Port 7687)<br/>Queries gene interactions"--> Neo4j:::readFlow
    NestJS --"TCP/HTTP (Port 8123/9000)<br/>Analytics queries"--> ClickHouse:::readFlow
    NestJS --"TCP (Port 5432)<br/>Validates session access"--> Postgres:::readFlow

    %% 4. Backend to Data Stores (Write/Mixed Flows - Orange)
    NestJS --"TCP (Port 6379)<br/>Cache checks and storage"--> Redis:::writeFlow
    NestJS --"TCP (Port 5432)<br/>Stores feedback & logs"--> Postgres:::writeFlow

    %% --- Styling Boundary ---
    style tdp_boundary fill:#f4faff,stroke:#0b4884,stroke-width:4px,stroke-dasharray: 5 5;
    style noteGSEA fill:#fff,stroke:#999,stroke-dasharray: 3 3;
    
