# **OptimistWeather**


## Abstract

---

This project focuses on optimizing weather query performance by leveraging **Redis** for caching and **PostgreSQL indexes** for 
efficient database operations. It is designed for students to gain hands-on experience with query optimization techniques and database performance tuning using Go.


## **Learning Objectives**

---

- Understand the role of **Redis** in caching and reducing query load.
- Learn how to use **PostgreSQL indexes** to optimize query performance.
- Explore error handling and monitoring in Go applications.


## **Context**

---

### Background
Weather APIs often experience performance bottlenecks due to frequent requests for real-time data. This project demonstrates how caching 
and indexing can significantly improve query efficiency and scalability.

### Real-World Applications
- Logistics and supply chain management
- Weather forecasting systems
- Disaster response platforms

### Technical Concepts
- **Caching:** Store frequently accessed data temporarily for quick retrieval.
- **Indexes:** Enhance database query performance by reducing the search space.

### Key Terms
- **Redis:** In-memory store used for caching and message brokering.
- **PostgreSQL Index:** A structure that accelerates data retrieval operations.


## **Resources**

---

- [Redis Documentation](https://redis.io/docs/)
- [PostgreSQL Indexing Guide](https://www.postgresql.org/docs/current/indexes.html)
- [Go Redis Client Library](https://github.com/go-redis/redis)
- [Cache strategies](https://medium.com/@mmoshikoo/cache-strategies-996e91c80303)



## **General Criteria**

---

- **Code Style:** Use `gofmt` for consistent formatting and adhere to Go idioms.
- **Compilation:** Ensure all code compiles cleanly using `go build`.
- **Execution:** Compatible with Linux or macOS systems; include Docker support.
- **Allowed Packages:**
    - `github.com/go-redis/redis/v8`
    - `github.com/lib/pq`
- **Disallowed Packages:** Frameworks that overly abstract caching or database operations.

### Build Command:
```bash
docker-compose up -d --build
```


## **Mandatory Features**

---

### **Outcomes**:

- Reduce database query load by caching frequently accessed weather data. 
- Functionality: Store and retrieve weather data in Redis with configurable TTL (Time-To-Live) / LRU / LFU.
- Optimize database queries by creating and using PostgreSQL indexes.
- Create indexes on relevant columns to improve query execution speed.


### Notes:

---

- Input: Location (e.g., city name or coordinates).
- Output: Weather data in JSON format.
- Handle invalid inputs gracefully and log errors.
- Ensure cache expiration for data freshness.


## Constraints:

---

- Handle errors gracefully and return appropriate HTTP status codes.
- All actions and errors must be logged.


## Support Section

---

- You can do it

### Debugging Tips:
- Use redis-cli to monitor cache hits and misses.
- Run EXPLAIN/EXPLAIN ANALYZE in PostgreSQL to analyze query plans.
```sql
EXPLAIN ANALYZE SELECT * FROM weather_data WHERE city = 'Astana';
```


### Common Issues and Fixes:

- Cache misses: Verify key structure and TTL configuration.
- Slow queries: Check if indexes are missing or misconfigured. 


### Where to Seek Help:
- Go forums and documentation
- Redis and PostgreSQL communities
- Stack Overflow for debugging Go applications



## API Endpoints

---

- TODO fill after

## This project has been created by:

Beknur Karshyga

Contacts:
- Email: [karshyga.beknur@gmail.com](mailto:karshyga.beknur@gmail.com)
- GitHub: [mrbelka12000](https://github.com/mrbelka12000)
- LinkedIn: [Beknur Karshyga](https://www.linkedin.com/in/beknur-karshyga/)

