# Design a URL shortener

## _Fun. / Non-Fun. Requirements
### Functional Requirements

1. Generate a unique short URL for a given long URL.
2. Redirect to the original URL when a short URL is accessed.
3. Allow user to customise their short URL (Optional).
4. Support link expiration, after which the short URL will not redirect to the original URL.
5. Track the number of times a short URL is accessed.
6. Analytics for a short URL, such as a number of times it was accessed, geographical location, browsers, etc.

### Non-Functional Requirements

1. High availability.(The service should be up 99.9% of the time)
2. Low latency. (The redirection to the original URL should happen in real-time)
3. Scalability (The system should be able to handle a large number of new URL shortenings and redirections)
4. Durability (Shortened URLs should not be lost)
5. Security to prevent malicious use, such as phishing attacks.

## _Traffic Estimation and Data Calculation_
#### Assumptions
1. 500M new URL shortenings per month.
2. read:write ratio is 100:1.

#### throughput requirement

* write URL per second = 500M / (30 days * 24 hours * 3600 seconds) = ~200 URLs/s
* read URL per second = 200 * 100 = 20K URLs/s

#### storage estimation

* For each shortened URL, we need to store 
   * the short URL : 7 character(Base62 encoded)
   * the original URL : 100 character(avg)
   * the creation date : 8 bytes (timestamp)
   * the expiration date: 8 bytes (timestamp)
   * click count : 4 bytes (integer)
* Total: 7 + 100 + 8 + 8 + 4 = 127 bytes
* storage requirement for one year: 
   * Total URLs for one year = 500M * 12 = 6B
   * Total storage = 6B * 127 bytes = 762 GB


## _API Design_

      
## _High-Level Architecture_
### Key Components



### _End-to-End Request Flow_


### high level design
![high level design](./images/URL_Shortening_System_Design.png)

### Database Design
![Database Design](./images/DistributedJobScheduler-Db-Design.png)

### _Deep Dive into Key Components_
