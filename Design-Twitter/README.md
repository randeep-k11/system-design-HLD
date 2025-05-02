# Design Twitter

## _Fun. / Non-Fun. Requirements_
### Functional Requirements
1. Create an account and login
2. Create, Edit and Delete a tweet
3. Follow other users
4. View a timeline of tweets from followed users
5. Like, Reply and retweet tweets
6. Search for tweets and users

### Non-Functional Requirements

1. Scale to 100+ million of users
2. Handle a high volume of tweets, likes and retweets
3. Highly available (99.999% uptime)
4. Security and Privacy of user data
5. Low latency for real-time interactions

## _Traffic Estimation and Data Calculation_
#### Assumptions
* Let us assume
   ```text
          total users: 1 billion 
          daily active users (DAU): 20% of total users = 200 million
          average tweets per user per day: 5
          200 million * 5 = 1 billion tweets per day
          request per second (RPS): 1 billion / 86400 seconds = 10^9 / 10^5 = 10k requests per second
   ```
#### Data Storage
* Lets assume average tweet size is 100 bytes
   ```text
   Storage capacity = 1 billion tweets/day * 100 bytes
                    = 1 * 10^9 * 100 bytes
                    = 1 * 10^9 * 10^2 bytes
                    = 100GB/day 
   ```
## _API Design_
 
      
## _High-Level Architecture_
### Key Components

1. Tweet Crud Service
   * Handles creating, reading and deleting tweets.
   * Write to tweet content DB. 
   * Publishes a message to queue (eq: kafka) for fanout processing.
   * Feeds into: 
     * Timeline fanout Service
     * Search Service (via CDC)
2. Reply CRUD Service
   * Manages replies to tweets.
   * Interacts with a replica db.
   * Also Published CDC events to be consumed downstream.
3. Search Service :queries Elasticsearch for tweet/search functionality
4. Timeline Service
   * Read User timeline.
   * Aggregates data from: 
       * Timeline Cache(redis)
       * Tweet DB
5. Timeline Fanout Service
   * Handle the fanout of tweets to followers.
   * Consumes messages from the Tweet CRUD queue and writes to the timeline cache.
   * fanout-on-write strategy.
       * when a tweet is created, it is sent to all followers of the user.
   * Timeline Cache
       * Redis or Memcached for fast access to user timelines.
       * Stores the latest tweets for each user.
* Profile Service
  * Handles profile-related features. 
  * Read from:
        * User Data (e.g., name, bio, picture)
        * Following Data (e.g., follow relationships)
### _End-to-End Request Flow_
* User submits a tweet.
* API Gateway routes to Tweet CRUD.
* The Tweet is written to DB.
* The Event is published to Message Queue.
* Fanout service reads from queue → pushes tweet to followers’ timelines.
* Timeline cache is updated for fast reads.
* Simultaneously, Search Service picks up the change and updates Elasticsearch.


### high level design
![high level design](./images/Twitter_System_Design.png)

### Database Design

### Questions
1. how to solve a celebrity problem?
   * The "celebrity problem" in a Twitter-like system refers to the scalability challenge when a user with millions of followers (a celebrity) posts a tweet. With a fanout-on-write approach, this tweet would need to be copied into the timelines of millions of followers, which can:
       * Overload the timeline fanout service
       * Cause high latency
       * Risk data loss if queues overflow
       * Be expensive in terms of compute and storage
   * Solutions to the Celebrity Problem
      1. Hybrid Fanout Strategy
          * Use fanout-on-write for normal users, but switch to fanout-on-read for celebrities.
                * Fanout-on-read means:
                       * No push to followers’ timelines.
                       * Tweets are fetched on-demand from the tweet store and merge with the timeline.

     2. Pull-based Timeline for Celebrity Tweets
          * Instead of pushing to all timelines:
                * Store tweets in a dedicated "celebrity tweet pool."
                * When a user opens their timeline, pull the latest tweets from this pool and merge with cached/personal timeline.
