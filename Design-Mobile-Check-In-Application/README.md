# Design Shazam Like Application with Song Identification

## _Fun. / Non-Fun. Requirements_

### Functional Requirements

* User Check-in
    * User can select a location and check-in, providing metadata such as a star rating.
    * The app records user location, time and duration of stay.
* NearBy Recommendations
    * Suggest nearby point of interest(POI's) based on user's current location.
* Batch upload of locations
    * Support importing location data with metadata in bulk.
* Reports
    * Top N categories with the highest check-ins(daily/monthly/yearly).
    * Most popular places (by check-ins) in a category(daily/monthly/yearly).
    * Best rated places in a category.

### Non-Functional Requirements

* **Highly available:**
  The system should be operational and accessible.
* **Performance:**
  Real-time recommendation and efficient report generation.
* **Scalability:**
  Handle a growing number of users and check-ins.
* **Reliability:**
  Ensure accurate data storage and retrieval

## _Traffic Estimation and Data Calculation_

#### Assumptions and storage

1. lets assume total users are 100 million and 10% DAU
    * DAUs = 10M
    * Each user checks in 2 times a day
    * Total check-ins = 10M * 5 = 50M
    * QPS = 50M / 24*60*60 = 50M/86400 = 578 QPS
2. Assume 10000 locations are uploaded daily in batch processing and each location metadata size is 2KB
    * Total data uploaded daily = 10000 * 2KB = 20MB/day
3. Analytics data and reports
    * let's Assume aggregated for top N categories and location ~= 1MB/day
4. Total storage
    * Check-ins = 50M
    * Locations = 20MB
    * Reports = 1MB
    * Total storage = 50M * (8 bytes/userID + 8 bytes/locationID + 8 bytes/time + 8 bytes/duration) + 20MB + 1MB = 50M *
      32 bytes + 21MB = 1.6GB + 21MB = 1.6GB + 0.021GB = 1.621GB = ~ 2GB/day

## _API Design_

1. 

## _High-Level Architecture_

### Key Components

1. **Batch Processing Service:** The Batch Processing Service accepts files containing location data (CSV/JSON),
   validates the data, processes it, and updates the Location Service database. It is designed to handle large-scale
   uploads efficiently and without disrupting the system's real-time operations.
    * Step 1: File Upload
        * The admin uploads a batch file containing location data via the admin panel or API.
        * the file is sent to the Batch Processing Service for validation and processing.
    * Step 2: file Validation
        * The Batch Processing Service validates the file format, size, and metadata.
        * It checks for duplicate entries, missing fields, and other errors.
    * Step 3: Asynchronous Processing
        * Once validated, the file is queued for processing to avoid blocking other operations:
            * Uses Kafka message queue to break the file into smaller chunks, where each chunk represents a subset of
              locations.
            * Each chunk is sent to worker nodes or microservices for parallel processing.
    * Step 4: Chunk Processing
        * Each worker processes a chunk of data:
            * Deduplication:
                * Check if the location already exists in the database (based on unique fields like latitude, longitude,
                  and name).
                * Update existing records if necessary or create new ones.
            * Database Interaction:
                * Interact with the Location Service to store or update location metadata in the database.
                * Geospatial indexing (e.g., using PostGIS)
    * Step 5: Error Handling
        * If errors are encountered (e.g., invalid records), these are logged and moved to a dead-letter queue for
          further analysis.
        * Notify the admin of records that failed validation or processing.
    * Step 6: Completion and Reporting
        * Once all chunks are processed, the Batch Processing Service sends a completion message to the admin.
        * Generate reports on the number of successful uploads, failed records, and other metrics.
        * the admin is notified of the result via mail or admin panel
2. **Check-In Service:** The Check-In Service is the core component responsible for recording and managing user
   check-ins at various locations. It ensures accurate data capture, updates relevant metrics, and integrates with other
   services like analytics and recommendations.
    * Step 1: User Initiates a Check-In
        * The user selects a location from nearby recommendations or searches for a location.
        * They provide optional metadata such as a star rating or a review.
        * The app sends a request to the Check-In Service.
    * Step 2 : Request validation and processing
        * The Check-In Service validates the request:
            * Ensure the location ID exists(via Location Service).
            * check if the user has already checked in at the location to avoid duplicates.
    * Step 3: Record the check-in:
        * A new check-in record is created in the database:
            * Field include: checkin_id, user_id, location_id, rating, checkin_time, metadata.
    * Step 4: Update Metrics
        * The check-In service publish an event to update metrics:
            * **Location Service:** Increment the check-in count for the location and average rating.
            * **Analytics Service:** Aggregate check-in data for reports and analytics.(eg: top N categories, popular
              places)
            * **User Service:** Update user check-in history and statistics.
            * **Recommendation Service:** Update user preferences and recommendations based on the check-in.
3. **Recommendation Service:** The Recommendation Service is responsible for providing personalized, location-based
   recommendations to users. Its primary goal is to suggest nearby locations (e.g., restaurants, parks) that match user
   preferences and contextual factors such as location, time, and past behavior.
    * Step 1: User Requests Recommendations
        * When the user opens the app, their current location (latitude and longitude) is sent to the Recommendation
          Service.
        * Optionally, the user can filter recommendations by:
            * Category (e.g., "restaurants").
            * Distance range (e.g., "within 5 km").
            * Other preferences (e.g., "highly rated" or "open now").
    * Step 2: Geospatial Query
        * The service queries the Location Service or the geospatial database for nearby POIs:
            * Filters based on proximity (e.g., POIs within a 5 km radius of the user).
            * Filters based on category, operating hours, and metadata.
    * Step 3: Scoring and Ranking
        * Each POI is scored based on various factors:
            * User preferences (e.g., past check-ins, ratings).
            * Popularity (e.g., total check-ins, average rating).
            * Distance (e.g.,Closer locations are ranked higher).
    * Step 4: Return Recommendations
        * A list of ranked POIs is returned to the user, including:
            * Name, category, distance, and rating.
            * Metadata like "Open now" or "Sponsored" tags.
    * Caching Layer: Frequently requested data (e.g., popular POIs in a region) is cached in Redis or Memcached for
      faster access.
4. **Homepage Service :** The Homepage Service is responsible for generating and displaying personalized, dynamic
   content on the app's homepage. This includes providing a snapshot of key features like nearby recommendations,
   check-in history, trending places, or user-specific updates (e.g., badges, reminders). It aggregates data from other
   microservices to create a cohesive and engaging experience.
    * Step 1: User Opens the App
        * The app makes a request to the Homepage Service with the user’s ID and current location.
        * Additional optional filters (e.g., preferences for categories) can be passed.
    * Step 2: Fetch Dynamic Content
        * The Homepage Service fetches data from various sources:
            * **Recommendation Service:** Nearby POIs and personalized suggestions.
            * **Check-In Service:** User’s recent check-ins and history.
            * **Analytics Service:** Trending places, top categories, and popular locations.
            * **User Service:** User-specific updates, badges, or notifications.
    * Step 3: Aggregate and Format Data
        * The service combines data from different sources into a unified view:
            * Organizes content into sections (e.g., "Nearby Recommendations," "Trending Places").
            * Formats data for display, including images, ratings, and metadata. example:
        ```
      {
            "nearby_places": [
              {
                "location_id": "12345",
                "name": "The Coffee House",
                "distance": "1.2 km",
                "rating": 4.5
              },
              {
                "location_id": "67890",
                "name": "Central Park",
                "distance": "2.1 km",
                "rating": 4.8
              }
             ],
            "trending_places": [
              {
                "location_id": "23456",
                "name": "Downtown Bistro",
                "category": "Restaurant",
                "checkins": 120
              }
             ],
            "recent_checkins": [
              {
                "location": "Starbucks",
                "checkin_time": "2024-12-27T12:00:00Z",
                "rating": 4.0
              }
             ],
             "user_highlights": {
                "badges": ["Explorer", "Top Reviewer"],
                "checkins_this_week": 5
               }
              }
       ```
      
5. **Analytics Service:** The Analytics Service is responsible for collecting, processing, and analyzing data related to user behavior and trends within the application. This service generates insights such as popular locations, trending categories, user engagement metrics, and detailed check-in reports.
    * Step 1: Data Collection
        * The service collects data from various sources:
            * **Check-In Service:** Records of user check-ins, including location, time, and metadata.
            * **Recommendation Service:** User preferences, recommendations, and interactions.
    * Step 2: Data Processing
        * The Analytics Service consumes the events and stores raw data in a data lake (amazon S3).
        * Aggregates data for reports (e.g., "Top 10 locations by check-ins last week").
        * Processed Data: Stored in a time-series database (e.g., InfluxDB) for quick access.
    * Step 3: Report Generation
       * Top Categories:
          * Aggregates check-in data by category (e.g., restaurants, cafes).
          * Calculates the highest check-in counts for daily, weekly, and monthly reports.
      * Top Locations:
          * Analyzes check-in and rating data to rank locations within a category.
          * Generates reports for popular and highly-rated locations.
      * User Reports:
         * Tracks user-specific data like check-in frequency, most visited categories, and engagement patterns.

### high level design

![high level design](./images/Mobile_Check-In_System_Design.png)

### Database Design

![DB design](./images/Shazam_DB_Design.png)

### _Questions_

1. 

* 