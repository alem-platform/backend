## Project Setup and Compilation
### Does the program compile successfully with `go build -o marketflow .`?
- [ ] Yes
- [ ] No

### Does the code follow gofumpt formatting standards?
- [ ] Yes
- [ ] No

### Does the program exit with a non-zero status code and clear error message when invalid arguments are provided?
- [ ] Yes
- [ ] No

### Does the program handle runtime errors gracefully without crashing?
- [ ] Yes
- [ ] No

### Is the program free of external packages except for database and cache operations?
- [ ] Yes
- [ ] No

## Architecture
### Is the application structured according to hexagonal architecture principles?
- [ ] Yes
- [ ] No

### Does the application have a clearly defined domain layer for core business logic?
- [ ] Yes
- [ ] No

### Does the application have a clearly defined application layer for use cases?
- [ ] Yes
- [ ] No

### Does the application implement proper adapter interfaces for web, storage, cache, and exchange?
- [ ] Yes
- [ ] No

### Are components properly decoupled with clear interfaces between layers?
- [ ] Yes
- [ ] No

## Data Acquisition
### Does the program correctly connect to exchange services on ports 40101, 40102, and 40103 in Live Mode?
- [ ] Yes
- [ ] No

### Does the program correctly handle data for all required currency pairs (BTCUSDT, DOGEUSDT, TONUSDT, SOLUSDT, ETHUSDT)?
- [ ] Yes
- [ ] No

### Does the program implement automatic reconnection when a connection fails?
- [ ] Yes
- [ ] No

### Does the program implement a functional Test Mode that generates synthetic market data?
- [ ] Yes
- [ ] No

### Can the program switch between Live Mode and Test Mode via API endpoints?
- [ ] Yes
- [ ] No

## Concurrency Implementation
### Does the program implement the Fan-in pattern to aggregate multiple data streams?
- [ ] Yes
- [ ] No

### Does the program implement the Fan-out pattern to distribute processing?
- [ ] Yes
- [ ] No

### Does the program implement a Worker Pool with appropriate workers for each data source?
- [ ] Yes
- [ ] No

### Does the program implement the Generator pattern for Test Mode data?
- [ ] Yes
- [ ] No

### Does the program batch data updates before inserting into PostgreSQL?
- [ ] Yes
- [ ] No

## Data Storage and Caching
### Does the program correctly store data in PostgreSQL with all required fields?
- [ ] Yes
- [ ] No

### Does the program correctly cache recent price data in Redis?
- [ ] Yes
- [ ] No

### Does the program maintain minute-based price averages, minimums, and maximums?
- [ ] Yes
- [ ] No

### Does the program correctly handle Redis failure by falling back to PostgreSQL?
- [ ] Yes
- [ ] No

### Does the program clean up irrelevant data from Redis?
- [ ] Yes
- [ ] No

## API Implementation
### Does the program implement all required GET endpoints for latest prices?
- [ ] Yes
- [ ] No

### Does the program implement all required GET endpoints for highest prices?
- [ ] Yes
- [ ] No

### Does the program implement all required GET endpoints for lowest prices?
- [ ] Yes
- [ ] No

### Does the program implement all required GET endpoints for average prices?
- [ ] Yes
- [ ] No

### Does the program implement mode switching endpoints (POST /mode/test and POST /mode/live)?
- [ ] Yes
- [ ] No

### Does the program implement a health check endpoint (GET /health)?
- [ ] Yes
- [ ] No

## Configuration and Logging
### Does the program properly read configuration from a configuration file?
- [ ] Yes
- [ ] No

### Does the configuration include PostgreSQL connection details?
- [ ] Yes
- [ ] No

### Does the configuration include Redis connection details?
- [ ] Yes
- [ ] No

### Does the configuration include exchange connection details?
- [ ] Yes
- [ ] No

### Does the program use log/slog for appropriate logging?
- [ ] Yes
- [ ] No

### Do logs include contextual information like timestamps and IDs?
- [ ] Yes
- [ ] No

## System Operation
### Does the program implement graceful shutdown handling?
- [ ] Yes
- [ ] No

### Does the program display usage information with `--help` flag?
- [ ] Yes
- [ ] No

### Does the program handle signals like SIGINT and SIGTERM properly?
- [ ] Yes
- [ ] No

## Project Defense

### Can the team explain their database design decisions?
- [ ] Yes
- [ ] No

### Can the team explain how they implemented complex queries?
- [ ] Yes
- [ ] No

### Can the team demonstrate understanding of PostgreSQL features used?
- [ ] Yes
- [ ] No

### Can the team explain their transaction handling approach?
- [ ] Yes
- [ ] No

## Detailed Feedback

### What was great? What you liked the most about the program and the team performance?

### What could be better? How those improvements could positively impact the outcome?
