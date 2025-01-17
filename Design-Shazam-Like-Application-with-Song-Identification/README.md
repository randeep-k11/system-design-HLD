# Design Shazam Like Application with Song Identification

## _Fun. / Non-Fun. Requirements_
### Functional Requirements
* Efficiently preprocess and store fingerprints for millions of songs.
* As a user, I should be able to record an audio and the system should identify return the song from the audio.

### Non-Functional Requirements
* **Highly available:**
  The system should be operational and accessible whenever users attempt to identify a song.
* **Low latency:**
  Low latency in Shazam's system means that the time from when a user records a sample to when they receive identification should be minimal
* **Scalability:**
  Shazam must handle a large and potentially unpredictable number of requests from users around the world.
  scalability ensures that as the user base grows and request volume increases, the system can handle the additional load without performance degradation.
* **Robustness:** Shazam's system should handle and recover from various kinds of failures, such as incorrect song samples, noisy environments, or partial system outages, without crashing or losing functionality.
  It should also manage a wide variety of music and audio quality and be able to accurately identify songs despite these variables.


## _Traffic Estimation and Data Calculation_
#### Assumptions and storage
1. Assume that there are around 100 million songs and the size of per song is 5MB and its metadata is 1KB.
   => 100M * 1KB = 100 * 10^6 * 1*10^3 = 100 * 10^9 = 100GB
2. There is no need to store the songs as it could lead to additional storage as well as compliance concerns regarding the rights of the songs.
   Regarding the storage of the metadata, it will require around 100GB (1KB * 100 million songs) of storage.
3. let's assume each song can produce 30 fingerprints of 64 bits each
   Size of each fingerprint: 64 bits = 8 bytes
   For 100 million songs:
   Total fingerprints=100M * 30 =3 billion fingerprints
   30 fingerprints of 8 bits each = 30 * 8 = 1920 bits = 240 bytes
   => 100M * 240 bytes = 100 * 10^6 * 240 = 24 * 10^9 = 24GB
4. QPS
   * Let's assume that we have a total of 100 million users.
   * DAU is 20% of total users, and each user makes 10 requests per day.
   * Total requests per day = 20M * 10 = 20M * 10^1 = 200M
   * Total requests per second = 200M / (24 * 3600) = 200M / 86400 = 200M / 10^5 = 2000 QPS

## _API Design_

1. **Identify the song**
   * Endpoint: POST v1/audio/
   * Request Body: 
     * form-data 
     * Key: file 
     * Value: (Audio file)
   * Response:
     ```json
        {
             "song": {
                     "title": "Blinding Lights",
                     "artist": "The Weeknd",
                     "album": "After Hours",
                     "release_date": "2019-11-29",
                     "duration": "3:22",
                     "genre": "Synthwave, Pop"
                    },
          "links": {
                     "spotify": "https://open.spotify.com/track/0VjIjW4GlUZAMYd2vXMi3b",
                     "apple_music": "https://music.apple.com/track/0VjIjW4GlUZAMYd2vXMi3b",
                     "youtube": "https://www.youtube.com/watch?v=fHI8X4OXluQ"
                 }
     }
     ```

## _High-Level Architecture_
### Key Components
1. Raw Audio Processing: Audio is stored in S3 and sent to the preprocessing service via Apache Flink.
2. Fingerprint Generation: Fingerprints are processed in Apache Flink and extract 20–30 unique fingerprints from each song published to Kafka. 
3. Storage and Processing: 
   * Fingerprints are consumed by two consumers:
        * ElasticSearch Consumer writes fingerprints into ElasticSearch for fast retrieval during song recognition.
        * MongoDB Consumer stores the fingerprints in MongoDB, likely for backup, analytics, or transactional consistency. 
   * Song metadata is consumed by metadata consumer which stores metadata in MongoDB:
        * Metadata includes song title, artist, album, genre, and other relevant information.
4. ElasticSearch: An inverted index is highly useful in the fingerprint matching process for Shazam because it enables efficient lookup of potential matches for the user's fingerprint.
   * An inverted index is a data structure that maps unique keys (in this case, fingerprints) to a list of associated identifiers (here, SongIDs).
     Structure of an Inverted Index for Shazam:
      * Key: A fingerprint hash (unique audio signature generated during preprocessing).
      * Value: A list of SongIDs that contain this fingerprint.
          ```
           FingerprintHash -> [SongID1, SongID3]
           FingerprintHash -> [SongID2]
           FingerprintHash -> [SongID1, SongID4, SongID5]
          ```
   * Why is an Inverted Index Useful for Shazam?
      * When a user submits an audio snippet, the system generates a single fingerprint for it. To identify the song, the backend must find all songs containing this fingerprint from potentially millions of songs in the database. An inverted index makes this lookup efficient and scalable.
      * Without an Inverted Index:
         * The system would have to scan through all fingerprints in the database, comparing the user's fingerprint with each one.
         * This is highly inefficient and scales poorly as the database grows.
      * With an Inverted Index:
         * The system directly queries the inverted index with the user fingerprint hash.
         * It retrieves only the relevant SongIDs, skipping irrelevant data.
         * The complexity of the lookup is significantly reduced.
   * How Does It Work in Shazam?
       * Step-by-Step Workflow:
          * Preprocessing Phase (Building the Index):
             * For each song:
                 * Generate 20–30 fingerprints using the "processing" algorithm.
                 * Add each fingerprint to the inverted index with the associated SongID.
                   Example:
                    ```
                     For SongID1, fingerprints: Hash1, Hash2, Hash3...hash30.
                     For SongID2, fingerprints: Hash31....Hash60.
                   
                     Resulting Index:
                        Hash1 -> [SongID1]
                        Hash2 -> [SongID1]
                        Hash3 -> [SongID1]
                        Hash40 -> [SongID2]
                   ```
          * Matching Phase (Querying the Index):
                  
              * The backend generates fingerprints for the user’s 5–10 second audio snippet. Let's assume the system extracts fingerprint: FP123
                * Query the Inverted Index for Each Fingerprint.
                * Search Fingerprint in elastic search DB:
                  * Exact Match: If hash1234 exists, the match is returned immediately.
                  * Fuzzy Match: If no exact match is found, the system searches for fingerprints within a Hamming distance or similarity score threshold.
                     ```lua
                        Hamming Distance:
                        Compare fingerprints at the bit level.
                        Example: FP123 (01100101) vs. FP124 (01100100) → Hamming distance = 1.
                        Set a threshold (e.g., Hamming distance ≤ 2) to tolerate small variations.
                     ```
            * Rank Matches:
                * Matches are ranked by their similarity score.
                * Example.
                  ```lua
                  Song A: 95% match
                  Song B: 92% match
                  Song C: 85% match
                  The highest-scoring match is returned
                  ```
                   
5. Recognition Service: 
   * **Functionality:**
      * Receives audio snippets from users.
      * call the **AudioPreprocessing Service** to generate fingerprint.
      * Once unique fingerprint is available, it calls **fingerprint matching service**.
      * fingerprint matching service first check in the redis cache if fingerprint found it returns the songId otherwise it queries the ElasticSearch inverted index to find candidate song.
      * Recognition Service receives the songId and calls **metadata service**.
      * Metadata service first check in the redis cache, if fingerprint found it returns, otherwise queries to metadata DB and returns the song details to the Recognition Service.
      * Returns the matched song to the user.
   * **Components:**
      * **Audio Fingerprinting Module:**
         * Generates unique fingerprints for audio snippets.
         * Converts audio signals into compact representations.
      * **Matching Algorithm:**
         * Compare user fingerprints with song fingerprints.
         * Calculates confidence scores for each candidate song.
      * **Querying Module:**
         * Retrieves candidate songs from the ElasticSearch inverted index.
         * Fetches full song fingerprints for comparison.
      * **Result Generation:**
         * Select the best match based on confidence scores.
         * Returns the matched song to the user.
6. Matching Service :
    * **Functionality:**
        * Compare user fingerprints with song fingerprints.
        * Calculates confidence scores for each candidate song.
        * Select the best match based on the scores.
    * **Components:**
        * **Fingerprint Comparison:**
            * Compare user fingerprints with song fingerprints.
            * Identifies overlapping fingerprints.
        * **Scoring Algorithm:**
            * Calculates confidence scores based on the number of matches.
            * Consider temporal alignment and other heuristics.
        * **Selection Logic:**
            * Choose the song with the highest confidence score.
            * Applies additional heuristics if needed.
* Song MetaData Service: 
    * **Functionality:**
        * Stores metadata for each song.
        * Provides song details for matching and identification.
    * **Components:**
        * **Metadata Storage:**
            * Stores song metadata, such as title, artist, album, genre.
            * Associates metadata with song fingerprints.
        * **Metadata Retrieval:**
            * Fetches song metadata for matched songs.
            * Provides details to the Recognition Service.

### high level design
![high level design](./images/Shazam_System_Design.png)

### Database Design
![DB design](./images/Shazam_DB_Design.png)

### _Questions_
1.  When to Consider Elasticsearch?
    * While a database can handle this scale, Elasticsearch may still be necessary if:
        * Fuzzy Matching: Fuzzy matching is a technique used in search or comparison to find results that are approximately similar, rather than requiring an exact match.
                          If queries need to account for noise or near matches in fingerprints. 
        * Scalability: If the dataset grows significantly (e.g., billions of songs) or query throughput demands increase beyond the database's capacity. 
        * Complex Queries: If your queries involve advanced matching patterns or full-text search capabilities.
    * High-Level Steps of Song Matching with Fuzzy Matching
      * User Input:
           * A user records an audio snippet from a live concert. 
           * The snippet contains noise and might not perfectly match the studio version.
        * Fingerprint Generation:
           * The system generates a unique fingerprint for the snippet(e.g.: hash1234).
        * Search Fingerprint in elastic search DB:
           * Exact Match: If hash1234 exists, the match is returned immediately.
           * Fuzzy Match: If no exact match is found, the system searches for fingerprints within a Hamming distance or similarity score threshold.
              ```lua
                 Hamming Distance:
                 Compare fingerprints at the bit level.
                 Example: FP123 (01100101) vs. FP124 (01100100) → Hamming distance = 1.
                 Set a threshold (e.g., Hamming distance ≤ 2) to tolerate small variations.
              ```
           
        * Rank Matches:
           * Matches are ranked by their similarity score.
           * Example.
             ```lua
             Song A: 95% match
             Song B: 92% match
             Song C: 85% match
             The highest-scoring match is returned
             ```
       * Return Metadata:
           * Once a match is found, the corresponding metadata (e.g., song name, artist) is fetched and returned to the user.
2. How do you ensure that you can quickly respond? 10 million concurrent users on Friday and Saturday nights? 
   * Shard the Inverted Index:
      * Partition the inverted index by FingerprintHash to distribute the workload across multiple servers or databases.
      * Example: Use consistent hashing or range-based sharding to divide hashes across shards.
   * Multi-Region Deployment:
      * Deploy services (e.g., Recognition Service, Kafka, ElasticSearch) across multiple geographic regions. 
      * Use a global load balancer to route users to the nearest region.
   * Cache Hot Fingerprints:
      * Use an in-memory cache (e.g., Redis or Memcached) for frequently queried fingerprints.
      * Hot fingerprints often represent popular or trending songs.
   * Use Load Balancers:
      * Distribute incoming requests across multiple Recognition Service instances.
      * Deploy global load balancers (e.g., AWS ELB, Google Cloud Load Balancer) to route user requests to the nearest backend instance.
      * Use geo-distribution to reduce latency by placing servers closer to users.
   * Horizontal Scaling: 
        * Add more Recognition Service instances to handle an increased load.
        * Use auto-scaling groups to automatically adjust the number of instances based on traffic.
   * CDN for Static Content
       * Use a Content Delivery Network (CDN) for serving static content (e.g., app assets, song previews).
   * Monitoring and Observability
        * Real-Time Monitoring
           * Use tools like Prometheus, Grafana, or Datadog to monitor:
              * Query latency.
              * CPU and memory usage.
              * Error rates and dropped requests.
        * Alerting
             * Set up alerts for anomalies, such as:
             * High error rates.
             * Slow query times.
             * System resource exhaustion.
