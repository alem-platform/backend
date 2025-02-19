# whale-with-benefits

## Learning Objectives

- Docker
- Application configuration
- Documentation

## Abstract

In this project, you will integrate Docker, environment variables, and improved documentation into your existing Go application, ensuring a more production-ready setup. The focus is on best practices for containerization, configuration, and clear deployment instructions. This assignment will bring you one step closer to successfully deploying production-grade applications.

## Context

In real-world projects, you often need to modify or enhance existing applications. This is one such scenario.

### Docker and configurations in applications

#### Docker
Before Docker, developers and administrators struggled with deployment across different environments. Applications that worked on one machine often failed elsewhere due to varying dependencies, library versions, or system configurations.

Docker solves this problem by packaging an application with all its dependencies (libraries, configurations, system tools) into lightweight, isolated containers. This ensures a consistent runtime environment—if the application runs locally in a container, it will run the same way in development and production. Docker improves portability and deployment reliability.

#### Configuration files
Configuration files store application parameters separately from the code. This approach enhances flexibility and maintainability:

- **Modify settings without changing code** – Database URLs, API keys, and other parameters can be updated without recompiling.

- **Environment-specific configurations** – Different configurations for local, test, and production environments.

- **Reusability** – The same configuration can be shared across multiple deployments.

#### Environment variables

Environment variables dynamically pass configuration details to applications. Unlike configuration files, they are not committed to version control, making them ideal for sensitive data like API keys and database credentials.

This project is another step towards working with production-ready applications.

---
## Resources

- Read about Docker [here](https://docs.docker.com/)
- Read about Docker Compose [here](https://docs.docker.com/compose/)
- Read about Configuration file [here](https://en.wikipedia.org/wiki/Configuration_file)
- Read about environment variables [here](https://en.wikipedia.org/wiki/Environment_variable) and [here](https://gobyexample.com/environment-variables)

---

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- External packages are allowed only for working with the database. If you use any other external packages, you will receive a grade of `0`.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o <project name> <path>
```

- If an error occurs during startup (e.g., invalid command-line arguments, failure to bind to a port), the program must exit with a non-zero status code and display a clear, understandable error message.
  During normal operation, the server must handle errors gracefully, returning appropriate HTTP status codes to the client without crashing.

- If there is a `.env` file in the repository (in any commit) you will receive a grade of `0`.

---
## Mandatory Part

### Baseline

You will modify and improve the `1337b04rd` project. Your goal is to:

- Add configuration file support.
- Integrate `.env` files for sensitive data.
- Enable Docker usage by creating `Dockerfile` and `docker-compose.yml`.
- Provide clear documentation in `README.md` and `DEPLOYMENT.md`.

### Requirements

Your applications **must read configuration files and environment variables**.

- The `.env` file should store sensitive information.
- You must manually read environment variables using `os.Getenv` (avoid external libraries for this).
- A `.env.example` file must be included in the repository as a reference.


### Configuration File Details
The configuration should include:

- Application port
- `S3 storage` details (address, port, etc.) from the `1337b04rd` project
- At least **two bucket names** in the `S3 storage`
- External API details (if applicable)

### Dockerfile Requirements

- The project **must have a** `Dockerfile` that defines how the application runs inside a container.
- The `Dockerfile` **must be tailored to this specific project** (no general-purpose Docker setups).
- The final image **must not contain unnecessary dependencies or development tools**.

### Docker Compose Requirements

The project must have a `docker-compose.yml` file to manage services.

Required services:
- **Application container**
- **Database container** 
- **S3-compatible storage**

**All services must start with a single command** using docker-compose up.

---

### README.md
Your `README.md` should include:

- A **clear project description**
- Instructions on **how to configure** the application
- How to **run the project** (both manually and with Docker)

### DEPLOYMENT.md
Your `DEPLOYMENT.md` should provide:

- **Step-by-step deployment instructions**
- How to **set up environment variables**
- How to use **Docker and Docker Compose** to run the project

---

### Usage

Your project must be runnable with a single command:

```shell
docker-compose up --build
```

This command should build and start all required services automatically.

## Support

[😉](https://www.reddit.com/r/docker/comments/keq9el/please_someone_explain_docker_to_me_like_i_am_an/)

## Guidelines from Author

1. Add support for configuration files and environment variables.
2. Implement a project-specific `Dockerfile`.
3. Provide a `docker-compose.yml` file with required services.
4. Update `README.md` and `DEPLOYMENT.md` with clear instructions.

---

## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)
