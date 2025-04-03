# [SSH](https://www.snailbook.com/index.html)-[IM](https://en.wikipedia.org/wiki/Instant_messaging)

## Learning Objectives

- Concurrency and Parallelism
- SSH Protocol
- CLI and Terminal Interaction
- Authorization

## Abstract

In this project, you will create an **SSH chat application**, a terminal-based chat application accessible via the SSH protocol. Unlike web-based chat systems, SSH-IM operates entirely in the terminal, offering a lightweight, secure, and privacy-focused way for users to communicate. You'll build a robust system that supports multiple users, public and private messaging, message history, and secure authentication—all while learning key networking and programming concepts.

This project is ideal for understanding secure communication protocols, real-time systems, and concurrent programming in Go. By the end, you'll have a functional chat application that demonstrates practical skills in a developer-friendly environment.

## Context

SSH (**S**ecure **Sh**ell) is a protocol primarily used for secure remote access over unsecured networks. Beyond logging into remote systems, SSH’s encryption and authentication features make it versatile for applications like secure messaging. In SSH-IM, you'll leverage SSH to create a chat platform where users connect using standard SSH clients (e.g., `ssh` on the command line), authenticate with public keys, and interact in real time.

This project bridges networking fundamentals with interactive application design, focusing on security and concurrency within a terminal environment.

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
  During normal operation, the server must handle errors gracefully.

---
## Mandatory Part

### Baseline

Develop the `SSH-IM` application, a **secure shell chat** app built in Go. Focus on writing clean, maintainable code that meets the functional requirements below.

### Functional Requirements

#### SSH Server

The SSH server is the backbone of SSH-IM, enabling secure client connections and managing chat sessions.

##### Basic Server Setup

- **Port**: Listen on a configurable port (default: `2222`).
- **Concurrency**: Handle multiple client connections concurrently using goroutines.
- **Package**: Use `golang.org/x/crypto/ssh` for SSH functionality.

##### Authentication

- Support [public key](https://www.geeksforgeeks.org/difference-between-private-key-and-public-key/) authentication.
- Store user credentials securely (authorized keys).
- Implement new user registration with proper credential management.

- **Public Key Authentication**: Authenticate users using SSH [public keys](https://www.geeksforgeeks.org/difference-between-private-key-and-public-key/).
- **User Storage**: Store usernames and their associated public keys in a PostgreSQL database table (e.g., `users: id, username, public_key`).
- **User Registration**: Provide a mechanism to add new users. For simplicity, self-registration by clients is not required.
- **Process**:
  - When a client connects with `ssh username@localhost -p 2222`, the server checks if the provided public key matches one associated with username in the database.
  - If authenticated, start a chat session; otherwise, reject the connection.

##### Connection Management

- **Sessions**: Track active client sessions and detect disconnections.
- **Timeouts**: Disconnect idle clients after 5 minutes.
- **Error Handling**: Gracefully handle failed connections and log errors.

##### Security Configuration

- **Host Keys**: Generate and use valid SSH host keys for the server.
- **Ciphers**: Enforce strong ciphers and key exchange methods.
- **Rate Limiting**: Limit authentication attempts to prevent brute-force attacks (e.g., max 5 attempts per minute per IP).
- **Logging**: Log all authentication attempts (successful and failed) with timestamps and details.

#### Chat Functionality

The chat system enables real-time communication between connected users through the terminal.

##### Message Processing

- **Broadcasting**: Relay public messages to all connected users.
- **Private Messaging**: Allow users to send private messages to specific users.
- **History**: Store all messages (public and private) in a PostgreSQL database table (e.g., `messages: id, sender_id, message, timestamp, is_private, recipient_id`).
- **Username**: Derive the username from the authenticated SSH connection (based on the public key).
- **Limits**: Restrict messages to 200 characters, encoded in UTF-8.
- **Rate Limiting**: Enforce a limit of 10 messages per minute per user.
- **Commands**: Process inputs starting with `/` as commands; otherwise, treat them as public messages.

##### Message Formatting

- **Public Messages**: [timestamp] <username>: `message`
  - Example: [14:30:15] <alice>: `Hello everyone!`
- **Private Messages (Sender)**: `[timestamp] * To username: message`
  - Example: `[14:30:20] * To bob: How’s it going?`
- **Private Messages (Recipient)**: `[timestamp] <username> (PRIVATE): message`
  - Example: `[14:30:20] <alice> (PRIVATE): How’s it going?`
- **System Messages**: `[timestamp] * System: message`
  - Example: `[14:30:00] * System: Welcome alice! There are 3 users online.`

##### Chat Commands

Implement these commands, processed only for the issuing user:

- `/help`: List all available commands.
- `/list`: Display all currently connected usernames (show duplicates if a user has multiple sessions).
- `/history [N]`: Show the last N messages (default 20) that are either public or private involving the user (sender or recipient).
- `/msg <username> <message>`: Send a private message to `<username>`. If the user is offline, display an error like `* System: User not online`.
- `/quit` or `/exit`: Disconnect the user from the server.

#### Outcomes:

- A secure, SSH-based chat system with real-time messaging.
- Robust authentication and session management using public keys.
- Persistent message history in PostgreSQL.
- Comprehensive logging with `log/slog`.

#### Constraints

- Handle errors gracefully and ensure proper error messages for users.
- All actions and errors must be logged.

### Flags/Options Support

- `-p`, `--port PORT` - Specify port number (default: `2222`)
- `--help`: Display usage information.


### Usage

**Launching the app**

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
$ ssh username@localhost -p 2222
Welcome to SSH Chat!
[14:30:00] * System: Welcome username! There are 3 users online.
[14:30:05] <bob>: Hey, good to see you!
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

**Start Small**: Build a minimum viable product ([MVP](https://www.techopedia.com/definition/27809/minimum-viable-product-mvp)) with basic connectivity, then add features.  
**Test Thoroughly**: Try connecting from multiple machines or terminals to simulate real usage.  
**Focus on UX**: Ensure commands and messages are intuitive and well-formatted.  
**Iterate**: Revisit requirements if stuck and refine your code incrementally.

Good luck building SSH-IM! This project will sharpen your skills in secure networking and real-time systems.

---
## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)