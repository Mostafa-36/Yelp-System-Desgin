# Yelp — System Design

A system design case study for a Yelp-like platform that allows users to discover local businesses, read reviews, and write their own. This document captures the requirements, APIs, data model, high-level architecture, and proposed scalability deep dives represented in the design.

---

## Table of Contents

1. [Overview]
2. [Requirements]
3. [Core Entities and Data Model]
4. [API Design]
5. [High-Level Design]
6. [Deep Dives]

---

## 1. Overview

Yelp is an online platform that allows users to search for and review local businesses, restaurants, and services.

![System Overview](./screens/Yelp%20system%20desgin.png)

### Goals

- Search for businesses by name, location, and category.
- View business details and associated reviews.
- Submit a review with a required star rating and optional text.
- Support low-latency search alongside a large user and business population.

---

## 2. Requirements

### 2.1 Functional Requirements

1. Users can search for businesses by **name**, **location (latitude/longitude)**, and **category**.
2. Users can view business details and reviews.
3. Users can leave a review for a business. A review must contain a **1–5 star rating** and may contain optional text.

### 2.2 Non-Functional Requirements

| Requirement        | Target                                            |
| ------------------ | ------------------------------------------------- |
| **Search latency** | Less than 500 ms                                  |
| **Availability**   | High availability                                 |
| **Consistency**    | Eventual consistency is acceptable                |
| **Scalability**    | 100 million daily users and 10 million businesses |

### 2.3 Constraint

The design assumes that **each user can leave only one review per business**. The enforcement mechanism is discussed in the deep dives.

---

## 3. Core Entities and Data Model

The initial model contains three main entities: `User`, `Business`, and `Review`.

### 3.1 Business

Represents a business listed on the platform.

| Field           | Description       |
| --------------- | ----------------- |
| `businessId`    | Primary key       |
| `name`          | Business name     |
| `location.lat`  | Latitude          |
| `location.long` | Longitude         |
| `category`      | Business category |
| `numRatings`    | Number of ratings |
| `avgRatings`    | Average ratings   |

### 3.2 Review

Represents a user's review of a business.

| Field        | Description                             |
| ------------ | --------------------------------------- |
| `reviewId`   | Primary key                             |
| `businessId` | Foreign key / reference to the business |
| `userId`     | User who submitted the review           |
| `starRating` | Required rating from 1 to 5             |
| `text`       | Optional review text                    |

### 3.3 User

Represents a user who searches for businesses and submits reviews.

> _Note: The diagram identifies `User` as a core entity, but does not define its full schema._

### Data Integrity

To enforce the one-review-per-user-per-business rule, the persistent data model should enforce uniqueness for the combination of `userId` and `businessId`. Checking only in application code is insufficient to prevent concurrent duplicate submissions.

---

## 4. API Design

The following endpoints cover the core functional requirements.

### 4.1 Search Businesses

`GET /v1/businesses?query={query}&location={location}&category={category}&page={page}`

Searches for businesses using optional query parameters.

**Example Response**

```json
{
  "status": "success",
  "businesses": [
    {
      "businessId": "business-123",
      "name": "Example Cafe"
    }
  ]
}
```

_(The response example is illustrative. The exact fields returned by the search endpoint are not fully specified in the design.)_

### 4.2 View a Business

`GET /v1/businesses/{businessId}`

Returns business details and its reviews.

**Example Response**

```json
{
  "businessId": "business-123",
  "location": {
    "lat": "latitude",
    "long": "longitude"
  },
  "category": "cafe",
  "reviews": [
    {
      "reviewId": "review-123",
      "starRating": 5,
      "text": "Example review"
    }
  ]
}
```

### 4.3 Create a Review

`POST /v1/businesses/{businessId}/reviews`

**Example Request**

```json
{
  "starRating": 5,
  "text": "Great service and atmosphere."
}
```

_(The authenticated user's identity should be obtained from the authentication context rather than trusted from an arbitrary client-supplied `userId`. Authentication details are not specified in the original diagram.)_

**Example Response**

```json
{
  "businessId": "business-123",
  "reviewId": "review-123",
  "starRating": 5,
  "text": "Great service and atmosphere."
}
```

### API Notes

- Search results should be **paginated** since the result set can be large.
- Validate that `starRating` is an integer between 1 and 5.
- Define consistent error responses for invalid input, missing businesses, duplicate reviews, and unexpected failures.

---

## 5. High-Level Design

### 5.1 System Components

![High-Level Architecture](./screens/Yelp%20system%20HLD.png)

The system consists of the following components:

- **Client:** Provides the user interface through which users can search for businesses, view business details, and submit reviews.
- **API Gateway:** Acts as the entry point for incoming client requests. It handles authentication and authorization, enforces rate limiting, manages TLS termination, and routes requests to the appropriate backend services.
- **Business Server:** Handles business-related operations, including searching for businesses, retrieving business details, and querying associated information.
- **Review Server:** Handles review-related operations, including validating, submitting, and managing reviews associated with specific businesses.
- **Database (PostgreSQL):** Stores and manages business information, user reviews, and the relationships between businesses and reviews. PostgreSQL ensures data consistency and supports efficient querying and data retrieval.

---

### 5.2 Business Search

**Requirement:** Users should be able to search for businesses based on criteria such as name, location, and category.

#### Request Flow

1. **Client:** Sends a `GET` request to `/businesses`, including optional query parameters such as name, location, and category.

- _Example:_ `GET /businesses?name=restaurant&location=Cairo&category=food`

2. **API Gateway:** Applies the appropriate authentication and rate-limiting policies, then routes the request to the Business Server.
3. **Business Server:** Validates the search parameters and queries PostgreSQL for businesses matching the specified criteria.
4. **Database (PostgreSQL):** Executes the query and returns the matching business records.
5. **Business Server:** Processes the results and returns the response to the API Gateway.
6. **API Gateway:** Forwards the response to the client.

---

### 5.3 Viewing Business Details

**Requirement:** Users should be able to view detailed information about a specific business, including its associated reviews.

#### Request Flow

1. **Client:** Sends a `GET` request to `/businesses/{businessId}` to retrieve the details of the selected business.

- _Example:_ `GET /businesses/123`

2. **API Gateway:** Applies the appropriate authentication and rate-limiting policies, then routes the request to the Business Server.
3. **Business Server:** Validates the business ID and retrieves the business details from PostgreSQL.
4. **Database (PostgreSQL):** Returns the requested business information. If the reviews are stored in the same database, the server can also retrieve the associated reviews.
5. **Business Server:** Combines the required information into a response and sends it to the API Gateway.
6. **API Gateway:** Forwards the response to the client, which displays the business details and reviews.

---

### 5.4 Submitting Business Reviews

**Requirement:** Users should be able to submit reviews for specific businesses.

#### Introducing a Dedicated Review Server

To handle review submissions, we introduce a dedicated Review Server that is separate from the Business Server. This separation is motivated by the following considerations:

- **Separation of Concerns:** The Business Server focuses on business-related operations, while the Review Server handles review-related operations. This makes the system easier to maintain, develop, and scale.
- **Independent Horizontal Scaling:** The system is expected to support approximately 100 million daily users. However, not every user will submit a review; business searches and detail requests generate significantly more traffic than review submissions. By separating these workloads, we can scale the Business Server and Review Server independently according to their respective traffic patterns (e.g., allocating more Business Server instances to handle high search volume without scaling the Review Server at the same rate).
- **Resource Optimization:** Independent scaling avoids allocating unnecessary resources to services that do not require the same processing capacity, improving resource utilization and reducing infrastructure costs.

#### Request Flow

1. **Client:** Sends a `POST` request to `/businesses/{businessId}/reviews`, including the review content in the request body.

- _Example:_

```http
POST /businesses/123/reviews
Content-Type: application/json

{
  "rating": 5,
  "comment": "Excellent service and a great experience."
}

```

2. **API Gateway:** Authenticates the user, applies rate limiting, and routes the request to the Review Server.
3. **Review Server:** Validates the submitted review, verifies that the target business exists, and ensures the review is associated with the correct business and authenticated user.
4. **Database (PostgreSQL):** Stores the new review along with its associated `businessId`, `userId`, review content, rating, and creation timestamp.
5. **Review Server:** Confirms that the review has been successfully persisted and constructs the appropriate response.
6. **API Gateway:** Forwards the response to the client, confirming that the review was submitted successfully.

---

## 6. Deep Dives

![Final System Architecture](./screens/Yelp%20system%20Deep%20Dive.png)

### 6.1 How would you efficiently calculate and update the average rating for businesses to ensure it's readily available in search results?

**Problem:** Calculating the average rating dynamically on every query introduces computational complexity and increases latency.

**Solutions Considered:**

1. **Cron Jobs / Batch Processing**

- _Solution:_ Calculate and update the average rating periodically (every hour, day, etc.).
- _Drawback:_ Data is not real-time.

2. **Synchronous Calculation + Pessimistic Locking (Accepted)**

- _Solution:_ Add `numRatings` and `avgRatings` fields directly to the business table. Apply a lock during each review submission operation to calculate the new average quickly and keep it real-time.
- _Trade-off:_ Slightly increases latency on write/submission operations, which is an acceptable compromise for immediate consistency.

---

### 6.2 How can you improve search to handle complex queries more efficiently?

**Problem:** Using standard SQL queries for searching (especially with latitude/longitude filtering) is expensive and inefficient, even with spatial/composite indexes.

**Solutions:**

1. **Elasticsearch (Accepted):** Utilizes an **inverted index** to make text and filter searches extremely fast, specifically designed for this use case.

- _New Challenge:_ How are changes reflected in Elasticsearch?
- _Solution:_ **CDC (Change Data Capture)** to capture modifications in the primary database and sync them to Elasticsearch in near real-time.

2. **PostgreSQL Extension (PostGIS):** Extends PostgreSQL with native capabilities to handle complex Geospatial Data efficiently.

---

### Additional System Bottlenecks & Solutions

1. **Sudden Huge Load:** Scale horizontally (or vertically if a single powerful machine can handle it); utilize request queueing to prevent server overload if slight latency increases are acceptable.
2. **High Load on Database:** _(Optimize the database first)_

- **For Heavy Reads:** Caching, Read Replicas, CDN, and Proper Indexing.
- **For Heavy Writes:** Batching, Request Queuing, Sharding, and Asynchronous Processing.

---
