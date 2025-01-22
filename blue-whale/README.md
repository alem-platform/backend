# blue-whale

## Learning Objectives

- Docker
- Application configuration
- Documentation

## Abstract

In this project, you need to add Docker, configurations for your previous project(s).  

This assignment will move you one step closer to successfully deploying production-grade applications.

## Context

When working on a project at work or anywhere else, there are situations when something needs to be changed or added to the project. There is such a case now.

### Docker and configurations in applications

#### Docker:
Before Docker came along, developers and administrators often faced challenges when deploying and scaling applications across different environments. An application that ran perfectly on one machine could fail to run correctly on another due to differences in the environment, such as library versions, system components, or operating systems.

Docker proposes packaging an application together with all its dependencies (libraries, configurations, system tools) into lightweight, isolated containers. This ensures a consistent runtime environment: if the application works in the container locally, it will work the same way on a development server and in production. This approach simplifies portability and increases deployment reliability.

#### Configuration files:
Configuration files are documents that store information about an application’s parameters and settings. Instead of embedding these parameters directly into the code, they are specified in separate files. This improves flexibility and ease of updates:

- **Managing settings without changing the code:**
You can modify parameters (e.g., database addresses or API keys) without recompiling the application.

- **Adapting to different environments:**
For each environment (local machine, test, production), you can use separate configuration files.

- **Reusing settings:**
A single set of configurations can be applied to multiple versions of the application or different environments.

#### Environment variables:

Environment variables are dynamic values that can affect the behavior of running processes. They are used to pass configuration information to applications. Environment variables are set outside the application and can be accessed by the application at runtime.

This project is another small step towards a large and complex production project.

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
- External packages are allowed only for working with the database and `.env` files. If you use any other external packages, you will receive a grade of 0.
- The project(s) MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o <project name> .<path>
```

- If an error occurs during startup (e.g., invalid command-line arguments, failure to bind to a port), the program must exit with a non-zero status code and display a clear, understandable error message.
  During normal operation, the server must handle errors gracefully, returning appropriate HTTP status codes to the client without crashing.

- If there is a `.env` file in the repository (in any commit) you will receive a grade of `0`.

---
## Mandatory Part

### Baseline

Your task is to improve the project(s) `1337b04rd`, `triple-s` (if you did not use `minio` in `1337b04rd` project) by adding and integrating the configuration file(s) `.env`, enabling Docker and creating a comprehensive `README.md `and`DEPLOYMENT.md ` documentation.
### Requirements

Your applications have to read a configuration files and `.env` files for sensitive data. You can use this library [`github.com/joho/godotenv`](https://github.com/joho/godotenv) to read `.env` files.

The configuration file(s) must include:

- The application port
- The address and port of the `S3 storage` that you used in the `1337b04rd` project and other information if necessary
- The names of at least two buckets in the `S3 storage`
- A link to an external API and any required endpoints if necessary

Your need to add `dockerfile` files.

Your need to add `docker-compose` file.

The project should have cleanly written `README.md` and `DEPLOYMENT.md` files.

### Usage

In this task, you need to describe how to launch your application yourself.
All information about the deployment of the application should be stored in `DEPLOYMENT.md`.

**Important:** your project should be run by a single command.

## Support

[😉](https://www.reddit.com/r/docker/comments/keq9el/please_someone_explain_docker_to_me_like_i_am_an/)

## Guidelines from Author

1. Add support for configuration files and environment variables to your project.
2. Add `dockerfile` files.
3. Add `docker-compose` file.
4. Update the `README.md` and `DEPLOYMENT.md` files with the necessary information.

---

## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)
