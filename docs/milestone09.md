### **Formalize the URL Frontier as a Dedicated Service**

*   **Problem:** The current `Crawler Service` is simple and stateless, pulling work from a basic queue. This architecture lacks the intelligence to crawl the web efficiently. It cannot prioritize important pages, enforce "politeness" rules (i.e., not overwhelming a single web server), manage recrawl schedules based on update frequency, or de-duplicate newly discovered URLs at a massive scale.

*   **Solution:** Extract all crawl scheduling logic into a new, dedicated, stateful **URL Frontier Service**. This service becomes the central brain of the crawling operation.
    1.  When a `Crawler` service processes a page, it extracts all links and sends them as a "discovered URLs" batch to the `URL Frontier`.
    2.  The `URL Frontier Service` is responsible for storing, de-duplicating, prioritizing, and scheduling these URLs. It maintains a persistent database of all URLs.
    3.  Instead of reading from a simple queue, `Crawler` instances now actively request a "batch of work" (a list of URLs to crawl) directly from the `URL Frontier Service`, which dispenses them according to its internal scheduling and politeness logic.

*   **Trade-offs:**
    *   **Pros:**
        *   **Intelligent & Efficient Crawling:** Enables sophisticated prioritization (e.g., crawling important sites more frequently), dramatically improving index freshness and relevance.
        *   **Centralized Politeness:** Makes the crawler a better web citizen by enforcing domain-specific rate limits from a central point.
        *   **Massive URL Scalability:** Provides a dedicated, persistent storage solution designed to handle and de-duplicate trillions of URLs.
    *   **Cons:**
        *   **New Stateful Component:** The `URL Frontier` is a complex, stateful service with a high-performance database, becoming a new critical dependency in the data ingestion pipeline.
        *   **Increased Crawl Latency:** The crawl loop now involves more network calls (Crawler -> Frontier -> Crawler), which can slightly increase the latency for crawling a single page.
        *   **"Hot Spot" Risk:** The Frontier's database can become a performance bottleneck if not designed and scaled correctly.

---

### **Design the Architecture-as-Code (AaC)**

#### **Logical View (C4 Component Diagram)**

This diagram introduces the `URL Frontier` as the new central scheduler, replacing the simple message queue for crawl jobs.

```mermaid
graph TD
    subgraph Pharos Search System
        subgraph "Data Ingestion Flow"
            Crawler[Crawler Component] --"2.Requests crawl jobs from"--> UrlFrontier[URL Frontier Service]
            UrlFrontier --"3.Dispatches batch of URLs"--> Crawler
            Crawler --"1.Sends discovered URLs to"--> UrlFrontier
            
            %% Downstream Flow remains the same
            Crawler --"4.Saves content"--> ContentStore[("Content Store")]
            Crawler --"5.Publishes pointer"--> MessageQueue[("Message Queue")]
            MessageQueue --> Indexer[Indexer Component]
        end
    end

    %% Component Descriptions
    classDef component fill:#82a9d9,stroke:#333,stroke-width:2px;
    class Crawler,Indexer,UrlFrontier component;

    click Crawler "https://g.co/expl/gKEp" "Component: Crawler\nResponsibility: A stateless worker. Sends discovered URLs to the Frontier, requests new work from the Frontier, and processes the dispatched URLs."
    click UrlFrontier "https://g.co/expl/wZ9c" "Component: URL Frontier Service\nResponsibility: The stateful scheduler for the entire crawl operation. Manages URL storage, de-duplication, prioritization, and politeness."
```

---

#### **Physical View (Deployment Diagram)**

The physical view is updated to include the new `URL Frontier Service` container and its dedicated `URL Database`.

```mermaid
graph TD
    subgraph "Cloud Environment (e.g., AWS)" [Physical Boundary]
        
        subgraph "Container Orchestration Platform (e.g., Fargate/EKS)"
            CrawlerSvc[Crawler Service]
            IndexerSvc[Indexer Service]
            ApiSvc[API Service]
            RankingSvc[Ranking Service]
            UrlFrontierSvc[URL Frontier Service]
        end

        UrlDB[("URL Database\n(e.g., Amazon DynamoDB)")]
        %% Other managed services
        MessageBus[("Message Queue\n(AWS SQS)")]
        ObjectStore[("Object Storage\n(Amazon S3)")]
        SearchDB[("Managed Search Database\n(Amazon OpenSearch)")]

        %% New Crawl Loop
        CrawlerSvc --"Discovered URLs (gRPC/HTTP)"--> UrlFrontierSvc
        UrlFrontierSvc --"Requests work (gRPC/HTTP)"--> CrawlerSvc
        UrlFrontierSvc --"Reads/Writes URLs"--> UrlDB
        
        %% Existing Ingestion Flow
        CrawlerSvc --"Writes message"--> MessageBus
        IndexerSvc --"Reads message"--> MessageBus
    end
```

---

#### **Component-to-Resource Mapping Table**

The table is updated to include the new `URL Frontier Service` and its database.

| Logical Component            | Physical Resource / Technology                          | Rationale                                                                                                                                                             |
| ---------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Crawler** (Component)      | **Crawler Service** (Container on Fargate/EKS)          | Now a pure worker, decoupled from scheduling logic. Interacts with the Frontier service for work management.                                                          |
| **Indexer** (Component)      | **Indexer Service** (Container on Fargate/EKS)          | No changes. Continues its role as a stateless ETL service.                                                                                                            |
| **API** (Component)          | **API Service** (Container on Fargate/EKS)              | No changes. Continues to orchestrate the query flow.                                                                                                                  |
| **Ranking Service**          | **Ranking Service** (Container on Fargate/EKS)          | No changes. Continues to provide re-ranking logic.                                                                                                                    |
| **URL Frontier Service**     | **URL Frontier Service** (Container on Fargate/EKS)     | A new, stateful service that encapsulates all scheduling logic. Can be scaled independently to handle the load of managing URLs.                                    |
| **URL Database**             | **Amazon DynamoDB**                                     | **Massive Scalability & Performance.** DynamoDB is a managed key-value store that provides single-digit millisecond latency and virtually unlimited scale, which is essential for storing and querying billions of URLs by their hash. |
| **Other Managed Services**   | SQS, S3, OpenSearch                                     | No changes. These services continue to perform their established roles in the architecture.                                                                           |
