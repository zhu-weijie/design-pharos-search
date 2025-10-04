### **Project Requirements Document: Web Crawler and Search Engine**

*   **Version:** 1.0
*   **Date:** 4 October 2025

#### **1. Overview**

This document specifies the requirements for building a large-scale Web Crawler and Search Engine. The system's purpose is to systematically crawl the public web, process and index the discovered content, and provide a user-facing search interface to retrieve relevant web pages based on user queries.

The system will be architected as two primary, decoupled subsystems:

1.  **Web Crawler:** A distributed system responsible for fetching, storing, and managing web page content and metadata.
2.  **Search Engine:** A high-performance system responsible for indexing content provided by the crawler and serving low-latency search queries to users.

#### **2. Goals and Objectives**

*   **Primary Goal:** To provide users with a fast, accurate, and reliable way to find relevant information on the web.
*   **Technical Objective:** To build a highly scalable, available, and fault-tolerant distributed system capable of handling petabytes of data and thousands of queries per second.
*   **Strategic Objective:** To create a foundational data platform that indexes a significant portion of the web, enabling future data analysis and product opportunities.

#### **3. Success Metrics**

The success of this project will be measured by the following key performance indicators (KPIs):

*   **Search Quality:**
    *   **Click-Through Rate (CTR):** Percentage of users who click on one of the top 3 search results.
    *   **Query Abandonment Rate:** Percentage of search queries that result in no clicks, indicating irrelevant results.
*   **System Performance:**
    *   **Query Latency:** 95th percentile (p95) search query response time of less than 200ms.
    *   **Index Freshness:** Time from when a high-priority page (e.g., major news site) is updated to when it is reflected in search results.
*   **System Scale:**
    *   **Documents Indexed:** Total number of unique web pages successfully crawled and indexed.
    *   **Queries Per Second (QPS):** The peak number of search queries the system can handle while meeting latency SLOs.

---

#### **4. Functional Requirements (FRs)**

##### **Epic 4.1: Core Search Experience**

*   **FR1: Search Query Input**

    *   As a user, I want to enter keyword-based search queries into a text field so that I can find relevant documents.
    *   The system shall support multi-word queries.
    *   The system shall support exact phrase searches using double quotes (e.g., `"system design interview"`).

*   **FR2: Search Results Display**

    *   As a user, I want to view a list of ranked search results so that I can easily identify the most relevant pages.
    *   Each result item shall display the page **Title**, the **URL**, and a generated **Summary Snippet** containing the search keywords in context.
    *   The system shall paginate results, displaying a configurable number of results per page (e.g., 10).

*   **FR3: Relevance and Ranking**

    *   As a user, I want the most relevant results to appear at the top of the list.
    *   The ranking algorithm shall use multiple signals to determine relevance, including:
        *   **Link Authority:** The importance of a page based on the quantity and quality of other pages linking to it (e.g., PageRank).
        *   **On-Page Factors:** The presence, frequency, and location of query keywords in the page's title, headings, and body text.
        *   **Content Freshness:** The recency of the content.

##### **Epic 4.2: Web Crawler & Data Ingestion**

*   **FR4: URL Frontier Management**

    *   The system shall manage a queue of URLs to be crawled, starting from an initial list of **Seed URLs**.
    *   The URL frontier shall prioritize URLs based on factors like page importance and update frequency to ensure fresh and relevant content is crawled first.

*   **FR5: Content Fetching**

    *   The crawler shall fetch web page content over HTTP/HTTPS.
    *   The crawler shall identify itself with a clear `User-Agent` string.
    *   The crawler shall strictly adhere to rules specified in a website's `robots.txt` file.

*   **FR6: Link Extraction**

    *   The system shall parse HTML content to extract new hyperlinks.
    *   Relative URLs (e.g., `/about.html`) shall be converted to absolute URLs (e.g., `http://example.com/about.html`) and added to the URL frontier.

*   **FR7: Raw Content Storage**

    *   The crawler shall save the full, unprocessed HTML content of each successfully fetched page into a durable, highly-available object store.

##### **Epic 4.3: Indexing & Data Processing**

*   **FR8: Content Parsing and Cleaning**

    *   The system shall process raw HTML to extract clean, indexable text content.
    *   This process shall include removing HTML tags, scripts, styles, and non-ASCII characters.
    *   Common "stop words" (e.g., "the", "a", "is") shall be removed.

*   **FR9: Index Construction**

    *   The system shall construct a **Forward Index** (mapping from `docID` to words contained in it).
    *   The system shall construct an **Inverted Index** (mapping from a word to the list of `docID`s containing it) to enable fast query lookups.

*   **FR10: Duplicate Detection**

    *   The system shall implement URL normalization and hashing to detect and discard duplicate URLs in the frontier.
    *   The system shall use an algorithm (e.g., Simhash) to detect and prevent the indexing of near-duplicate web page content.

##### **Epic 4.4: System Administration & Monitoring**

*   **FR11: Monitoring Dashboard**

    *   As an administrator, I need a dashboard to monitor the health and performance of the system, including crawl rate, number of indexed documents, query latency, and error rates.

*   **FR12: Seed URL Management**

    *   As an administrator, I need a mechanism to add, remove, and update the list of seed URLs to guide the crawling process.

*   **FR13: Blocklist Management**

    *   As an administrator, I need the ability to maintain a global blocklist of domains or URL patterns that the crawler must not visit, regardless of `robots.txt` rules.

---

#### **5. Non-Functional Requirements (NFRs)**

##### **5.1 Performance**

*   **NFR1: Search Latency:** Search queries must have a p95 latency of **< 200 milliseconds** from the moment the request hits the server to the first byte of the response.
*   **NFR2: Throughput:** The search system shall be designed to handle an initial load of **1,000 queries per second (QPS)**, with the ability to scale beyond that.

##### **5.2 Scalability**

*   **NFR3: Data Volume:** The system's architecture must scale to store and index at least **10 billion web pages** (approx. 5 PB of text content).
*   **NFR4: Horizontal Scaling:** All system components (crawling, indexing, serving) shall be designed to scale horizontally by adding more machines. There shall be no single point of failure.

##### **5.3 Availability & Reliability**

*   **NFR5: Availability:** The user-facing search endpoint shall have a minimum uptime of **99.9%** (Service Level Objective).
*   **NFR6: Data Durability:** The raw content store and search indexes must be stored with a durability of **99.999999999%** (11 nines), protecting against data loss.

##### **5.4 Operational**

*   **NFR7: Content Freshness:**

    *   High-priority websites (e.g., top news sources) must be recrawled and re-indexed within **24 hours** of content changes.
    *   Lower-priority websites must be recrawled within **30 days**.

*   **NFR8: Politeness Policy:** The web crawler must enforce a configurable delay between consecutive requests to the same host/domain to avoid overloading web servers. A single crawler thread shall only make one request to a given host at a time.
*   **NFR9: Security:** The search query interface must sanitize all user inputs to prevent injection attacks.
*   **NFR10: Cost Efficiency:** The system shall use data compression for both the content store and the indexes to minimize storage costs.

---

#### **6. Out of Scope**

The following features will not be included in the initial version of this project:

*   Image, Video, or non-text search.
*   Near real-time indexing (e.g., live sports scores, breaking news within seconds).
*   Personalized search results based on user history or profile.
*   User accounts, saved searches, or authentication.
*   An advertising system.
