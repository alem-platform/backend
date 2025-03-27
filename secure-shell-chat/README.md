# [SSH](https://www.snailbook.com/index.html)-[IM](https://en.wikipedia.org/wiki/Instant_messaging)

## Learning Objectives

- Concurrency and Parallelism
- SSH Protocol
- CLI and Terminal Interaction
- Authorization

## Abstract

In this project, you will build an **SSH chat application**.

Similar projects are used by communities, development teams, and privacy-conscious users to communicate securely in terminal environments without relying on web-based interfaces. This application will create a robust chat system accessible via SSH protocol, allowing users to connect, exchange messages and use various commands for enhanced interaction. Additionally, it will manage user sessions, message history, and secure authentication.

This project will teach you how to implement secure network protocols, build interactive terminal applications, and develop real-time communication systems that are both privacy-focused and developer-friendly. You'll gain valuable experience in creating software that bridges the gap between networking fundamentals and practical user interaction.

## Context
SSH (**S**ecure **Sh**ell) is, first of all, a secure communication protocol for the secure operation of remote systems over an unsecured network. Although SSH is designed as a security tool, there are several ways to use it to bypass certain restrictions. Beyond basic remote access, SSH serves as a foundation for building various applications for secure communication, file exchange, and other collaborative tools. Developers can leverage SSH's encrypted tunneling capabilities to create secure messaging systems, file synchronization utilities, backup solutions, and collaborative development environments, all benefiting from SSH's robust authentication and encryption mechanisms.  

SSH Chat is a terminal-based real-time communication platform accessible via SSH protocol. 

The project is a secure, lightweight messaging system that demonstrates the basic principles of networking and concurrent programming principles.

---
## Resources
- Read about SSH Protocol [here](https://en.wikipedia.org/wiki/Secure_Shell#Standards_documentation).
- Read about Go SSH package [here](golang.org/x/crypto/ssh).

---

## General Criteria
- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- Only packages for working with the database, cache and `golang.org/x/crypto/ssh` are allowed. If you use any other external packages, you will receive a grade of `0`.
- The test coverage of your project's code must be at least 11%. A lower percentage means `0` points for your project.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o ssh-im .
```

- If an error occurs during startup (e.g., invalid command-line arguments, failure to bind to a port), the program must exit with a non-zero status code and display a clear, understandable error message.
  During normal operation, the server must handle errors gracefully, returning appropriate HTTP status codes to the client without crashing

---
## Mandatory Part

### Baseline
Your task is to develop the `SSH-IM` application, a **secure shell chat** app. It should be built using Go, emphasizing clean code and maintainability.

### Functional Requirements

#### SSH Server

The SSH Server is the core component of your application that enables secure connections from clients. It will authenticate users, manage sessions, and route messages between connected clients.

##### Basic Server Setup

- Implement an SSH server that listens on a configurable port (default: `2222`).
- The server should be able to handle multiple concurrent client connections.
- Your implementation must use the `golang.org/x/crypto/ssh` package.

##### Authentication

- Support [public key](https://www.geeksforgeeks.org/difference-between-private-key-and-public-key/) authentication
- Store user credentials securely (authorized keys)
- Implement a user registration mechanism for new users

##### Connection Management

- Handle client connections concurrently
- Manage active sessions and detect disconnections
- Implement proper error handling for failed connections
- Set appropriate timeouts for idle connections (5 minutes)

##### Security Configuration

- Generate and use proper SSH host keys
- Enforce strong ciphers and key exchange methods
- Implement rate limiting to prevent brute force attacks 
- Log authentication attempts (successful and failed)

**Connections must be handled efficiently. Use goroutines.**



#### Chat Functionality

Chat is the other main component of your `SSH-IM` application. It allows users to communicate with each other in real-time through the terminal interface. This section outlines how to implement robust messaging capabilities, chat commands, and user interaction systems.

##### Message Processing

- Implement a message broadcasting system to relay messages to all connected users
- Support private messaging between specific users
- Store message history for users to retrieve when they join
- The username provided as a command-line argument, extracted from the public key, or requested during connection.
- Messages limited to 200 characters per transmission
- UTF-8 encoding support for international character sets
- Support for command input prefixed with `/` character
- Implement appropriate message formatting for readability in the terminal
- Rate limiting of 10 messages per minute per user to prevent flooding

##### Chat Commands

- Create a command processor that handles special instructions prefixed with `/`
- Implement common chat commands such as:  
  `/help` - Display available commands    
  `/list` - Show currently connected users   
  `/history` - Command to see recent messages  
  `/msg` `<username>` `<message>` - Send private message  
  `/quit` or `/exit` - Disconnect from the server
  

##### Private Messages

When a private message is received, it should automatically appear in the terminal in a special format that distinguishes it from regular messages.  
Follow these requirements:

```
[17:52:03] <username> (PRIVATE): message content
```

This formatting makes it easy to distinguish private messages from public chat messages.

##### Viewing Past Private Messages

To view your private message history:

1. Use the `/history` command to see recent messages, including private ones.  
```
/history
```
2. To specify the number of messages to view:
```
/history 20
```

##### Sending Private Messages

To send a private message to another user:
```
/msg <username> Hello, how are you?
```

#### Outcomes:
- Establish a secure SSH-based chat system where users can connect and communicate in real time.
- Implement user authentication and session management using SSH keys.
- Store chat history in a database (PostgreSQL) for persistence and retrieval.
- Utilize Go's `log/slog` package for logging authentication attempts, message deliveries, and system events.

#### Constraints
- Handle errors gracefully and ensure proper error messages for users.
- All actions and errors must be logged.

### Flags/Options Support

- `-p`, `--port PORT` - Specify port number (default: `2222`)
- `-u`, `--user USERNAME` - Set username for the session


### Usage

**Launching the app**

Your program must be able to print usage information.

```sh
$ ./ssh-im --help
secure-shell-chat

Usage:
  ssh-im [--port <N>]  
  ssh-im  --help

Options:
  --help       Show this screen.
  --port N     Port number.
```

**Basic Connection**
```
$ ssh-im localhost -p 2222 -u alice
Connected to localhost
Welcome to SSH Chat!
[10:22:38] * System: Welcome alice! There are 5 users online.
[10:22:40] <bob>: Hi alice, welcome to the chat!
```

**Using Commands**

```
[10:22:38] <alice>: /help
[10:22:38] * System: Available commands:
  /help - Display available commands  
  /list - Show currently connected users  
  /msg <username> <message> - Send private message
  /quit - Disconnect from the server
  /exit - Disconnect from the server
```

**Private Messaging**

```
[10:22:38] <alice>: /msg bob Are you working on the new feature?
[10:22:38] * To bob: Are you working on the new feature?
[10:22:45] <bob> (PRIVATE): Yes, should be done by tomorrow.
```

**List Users**

```
[21:35:30] <alice> /list
=== Online Users ===
mrbelka12000
bob
alice
skantay
asari
Total online: 5
=== End of List ===

[21:35:40] <bob> /msg alice How's the project going?
[21:35:40] * To alice: How's the project going?
```

**Exit**

```
[10:22:38] <alice>: /quit
[10:22:38] * System: Goodbye!
Connection closed.
```

## Support

Look up materials with best practices for building a CLI application interface.

If you encounter a problem or confusion, head to Stack Overflow and GitHub.

---
## Guidelines from Author
When working on this project, pay close attention to interface design and user experience. Ensure that your SSH chat application is intuitive and robust, handling user input, authentication, and messaging efficiently.  
Test your implementation on multiple machines and network setups. If possible, collaborate with peers to simulate real-world usage scenarios. Before connecting, ensure that you know your local network IP address to facilitate testing.  
If you encounter challenges, revisit the project’s requirements and iterate on your implementation. Start with a [MVP](https://www.techopedia.com/definition/27809/minimum-viable-product-mvp) and refine it as you progress.

Good luck, and enjoy building your SSH-based chat system!

---
## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)