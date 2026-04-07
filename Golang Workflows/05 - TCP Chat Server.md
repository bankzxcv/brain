---
title: "Go Workflow: TCP Chat Server"
date: 2026-04-07
tags:
  - golang
  - workflow
  - project
  - networking
  - tcp
  - goroutines
  - channels
  - mutex
  - select
parent: "[[Golang Study]]"
---

# Go Workflow: TCP Chat Server

## Overview

Build a multi-client TCP chat server where users connect via telnet or netcat, choose a nickname, and communicate in real time. The server supports broadcast messages, private messages, and chat rooms -- all built with goroutines, channels, and mutexes.

**Concepts practiced:** `net` package (TCP listener, connections), goroutines, channels, `select`, `sync.Mutex`, maps, graceful shutdown with `context`, buffered I/O.

## Features

- Accept multiple concurrent TCP clients
- Broadcast messages to all connected clients
- Client registry with nicknames
- Private messages (`/msg nickname message`)
- Chat rooms (`/join room`, `/leave`, `/rooms`)
- Graceful shutdown that notifies clients

---

## Step-by-Step Implementation

### Step 1: TCP Server That Accepts Connections

Start with a bare TCP server that accepts connections and echoes back.

```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
)

func main() {
	listener, err := net.Listen("tcp", ":9000")
	if err != nil {
		log.Fatalf("Failed to start server: %v", err)
	}
	defer listener.Close()

	log.Printf("Chat server listening on :9000")

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Printf("Accept error: %v", err)
			continue
		}
		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	defer conn.Close()
	addr := conn.RemoteAddr().String()
	log.Printf("New connection from %s", addr)

	scanner := bufio.NewScanner(conn)
	for scanner.Scan() {
		line := scanner.Text()
		fmt.Fprintf(conn, "echo: %s\n", line)
	}

	log.Printf("Connection closed: %s", addr)
}
```

Test this with: `nc localhost 9000` or `telnet localhost 9000`.

> [!tip] Why `bufio.Scanner`?
> TCP is a stream protocol -- there are no built-in message boundaries. `bufio.Scanner` reads line-by-line (splitting on `\n`), which is the natural message boundary for a text chat protocol.

---

### Step 2: Handle Each Client in a Goroutine

Define a `Client` type that represents a connected user, with a channel for outgoing messages.

```go
import (
	"sync"
)

type Client struct {
	conn     net.Conn
	nickname string
	room     string
	outgoing chan string
}

type Server struct {
	mu       sync.RWMutex
	clients  map[string]*Client // nickname -> client
	rooms    map[string]map[string]*Client // room -> nickname -> client
}

func NewServer() *Server {
	return &Server{
		clients: make(map[string]*Client),
		rooms:   make(map[string]map[string]*Client),
	}
}
```

Each client gets a dedicated goroutine for writing outgoing messages. This prevents slow clients from blocking the sender.

```go
func (c *Client) writePump() {
	for msg := range c.outgoing {
		fmt.Fprint(c.conn, msg)
	}
}
```

> [!info] The Outgoing Channel Pattern
> Giving each client a buffered outgoing channel decouples "sending a message to a client" from "actually writing bytes to the TCP connection." If a client's connection is slow, messages queue in the channel instead of blocking the broadcaster.

---

### Step 3: Broadcast Messages to All Connected Clients

```go
func (s *Server) broadcast(senderNick, msg string) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	formatted := fmt.Sprintf("[%s] %s\n", senderNick, msg)
	for nick, client := range s.clients {
		if nick != senderNick {
			select {
			case client.outgoing <- formatted:
			default:
				// Client's buffer is full, skip to avoid blocking
				log.Printf("Dropping message to slow client: %s", nick)
			}
		}
	}
}
```

> [!warning] Non-Blocking Send
> The `select` with a `default` case makes the send non-blocking. If a client's outgoing channel is full (they are too slow to read), we drop the message rather than blocking the entire broadcast. This is a deliberate design choice to protect the server.

---

### Step 4: Client Registration with Nicknames

When a client connects, prompt them for a nickname before allowing chat.

```go
func (s *Server) registerClient(conn net.Conn) *Client {
	fmt.Fprint(conn, "Enter your nickname: ")
	scanner := bufio.NewScanner(conn)

	for scanner.Scan() {
		nick := strings.TrimSpace(scanner.Text())
		if nick == "" {
			fmt.Fprint(conn, "Nickname cannot be empty. Try again: ")
			continue
		}

		s.mu.Lock()
		if _, exists := s.clients[nick]; exists {
			s.mu.Unlock()
			fmt.Fprintf(conn, "Nickname '%s' is taken. Try another: ", nick)
			continue
		}

		client := &Client{
			conn:     conn,
			nickname: nick,
			room:     "general",
			outgoing: make(chan string, 256),
		}
		s.clients[nick] = client

		// Add to default room
		if s.rooms["general"] == nil {
			s.rooms["general"] = make(map[string]*Client)
		}
		s.rooms["general"][nick] = client
		s.mu.Unlock()

		return client
	}
	return nil
}

func (s *Server) removeClient(client *Client) {
	s.mu.Lock()
	defer s.mu.Unlock()

	delete(s.clients, client.nickname)

	// Remove from current room
	if room, ok := s.rooms[client.room]; ok {
		delete(room, client.nickname)
		if len(room) == 0 {
			delete(s.rooms, client.room)
		}
	}

	close(client.outgoing)
}
```

---

### Step 5: Private Messages

Parse commands that start with `/` and handle `/msg`.

```go
import "strings"

func (s *Server) handleCommand(sender *Client, line string) bool {
	parts := strings.SplitN(line, " ", 3)
	cmd := parts[0]

	switch cmd {
	case "/msg":
		if len(parts) < 3 {
			sender.outgoing <- "Usage: /msg <nickname> <message>\n"
			return true
		}
		s.privateMessage(sender, parts[1], parts[2])
		return true

	case "/join":
		if len(parts) < 2 {
			sender.outgoing <- "Usage: /join <room>\n"
			return true
		}
		s.joinRoom(sender, parts[1])
		return true

	case "/leave":
		s.leaveRoom(sender)
		return true

	case "/rooms":
		s.listRooms(sender)
		return true

	case "/who":
		s.listUsers(sender)
		return true

	case "/help":
		sender.outgoing <- `Available commands:
  /msg <nick> <message>  - Send a private message
  /join <room>           - Join a chat room
  /leave                 - Leave current room (back to general)
  /rooms                 - List active rooms
  /who                   - List online users
  /quit                  - Disconnect
  /help                  - Show this help
`
		return true

	case "/quit":
		sender.outgoing <- "Goodbye!\n"
		sender.conn.Close()
		return true
	}

	return false // not a command
}

func (s *Server) privateMessage(sender *Client, targetNick, msg string) {
	s.mu.RLock()
	target, ok := s.clients[targetNick]
	s.mu.RUnlock()

	if !ok {
		sender.outgoing <- fmt.Sprintf("User '%s' not found.\n", targetNick)
		return
	}

	formatted := fmt.Sprintf("[PM from %s] %s\n", sender.nickname, msg)
	select {
	case target.outgoing <- formatted:
		sender.outgoing <- fmt.Sprintf("[PM to %s] %s\n", targetNick, msg)
	default:
		sender.outgoing <- fmt.Sprintf("Could not deliver message to %s (buffer full).\n", targetNick)
	}
}
```

---

### Step 6: Chat Rooms

```go
func (s *Server) joinRoom(client *Client, roomName string) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// Remove from current room
	if room, ok := s.rooms[client.room]; ok {
		delete(room, client.nickname)
		// Notify room members
		for _, c := range room {
			select {
			case c.outgoing <- fmt.Sprintf("* %s left the room\n", client.nickname):
			default:
			}
		}
		if len(room) == 0 && client.room != "general" {
			delete(s.rooms, client.room)
		}
	}

	// Join new room
	if s.rooms[roomName] == nil {
		s.rooms[roomName] = make(map[string]*Client)
	}
	s.rooms[roomName][client.nickname] = client
	client.room = roomName

	// Notify new room members
	for nick, c := range s.rooms[roomName] {
		if nick != client.nickname {
			select {
			case c.outgoing <- fmt.Sprintf("* %s joined the room\n", client.nickname):
			default:
			}
		}
	}

	client.outgoing <- fmt.Sprintf("Joined room: %s (%d users)\n",
		roomName, len(s.rooms[roomName]))
}

func (s *Server) leaveRoom(client *Client) {
	if client.room == "general" {
		client.outgoing <- "You are already in the general room.\n"
		return
	}
	s.joinRoom(client, "general")
}

func (s *Server) listRooms(client *Client) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	var buf strings.Builder
	buf.WriteString("Active rooms:\n")
	for name, members := range s.rooms {
		marker := ""
		if name == client.room {
			marker = " (you are here)"
		}
		buf.WriteString(fmt.Sprintf("  #%s - %d users%s\n", name, len(members), marker))
	}
	client.outgoing <- buf.String()
}

func (s *Server) listUsers(client *Client) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	var buf strings.Builder
	buf.WriteString("Online users:\n")
	for nick, c := range s.clients {
		buf.WriteString(fmt.Sprintf("  %s (room: #%s)\n", nick, c.room))
	}
	buf.WriteString(fmt.Sprintf("Total: %d\n", len(s.clients)))
	client.outgoing <- buf.String()
}

// Room-scoped broadcast (instead of global broadcast)
func (s *Server) broadcastToRoom(senderNick, room, msg string) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	members, ok := s.rooms[room]
	if !ok {
		return
	}

	formatted := fmt.Sprintf("[#%s] [%s] %s\n", room, senderNick, msg)
	for nick, client := range members {
		if nick != senderNick {
			select {
			case client.outgoing <- formatted:
			default:
			}
		}
	}
}
```

---

### Step 7: Graceful Shutdown with Context

Wire everything together with proper shutdown handling.

```go
import (
	"context"
	"os/signal"
	"syscall"
)

func (s *Server) handleClient(client *Client) {
	defer s.removeClient(client)

	// Start write pump
	go client.writePump()

	// Announce join
	s.broadcastToRoom(client.nickname, client.room,
		fmt.Sprintf("* %s has connected", client.nickname))
	client.outgoing <- fmt.Sprintf("Welcome, %s! You are in #%s. Type /help for commands.\n",
		client.nickname, client.room)

	// Read loop
	scanner := bufio.NewScanner(client.conn)
	for scanner.Scan() {
		line := strings.TrimSpace(scanner.Text())
		if line == "" {
			continue
		}

		// Check if it's a command
		if strings.HasPrefix(line, "/") {
			if s.handleCommand(client, line) {
				continue
			}
		}

		// Regular message: broadcast to room
		s.broadcastToRoom(client.nickname, client.room, line)
	}

	// Announce leave
	s.broadcastToRoom(client.nickname, client.room,
		fmt.Sprintf("* %s has disconnected", client.nickname))
}

func (s *Server) shutdown() {
	s.mu.Lock()
	defer s.mu.Unlock()

	for _, client := range s.clients {
		client.outgoing <- "Server is shutting down. Goodbye!\n"
		client.conn.Close()
	}
}
```

---

## Full Solution

> [!success]- Full Solution
> ```go
> package main
> 
> import (
> 	"bufio"
> 	"context"
> 	"fmt"
> 	"log"
> 	"net"
> 	"os"
> 	"os/signal"
> 	"strings"
> 	"sync"
> 	"syscall"
> 	"time"
> )
> 
> // --- Types ---
> 
> type Client struct {
> 	conn     net.Conn
> 	nickname string
> 	room     string
> 	outgoing chan string
> }
> 
> func (c *Client) writePump() {
> 	for msg := range c.outgoing {
> 		c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
> 		fmt.Fprint(c.conn, msg)
> 	}
> }
> 
> type Server struct {
> 	mu      sync.RWMutex
> 	clients map[string]*Client
> 	rooms   map[string]map[string]*Client
> }
> 
> func NewServer() *Server {
> 	return &Server{
> 		clients: make(map[string]*Client),
> 		rooms: map[string]map[string]*Client{
> 			"general": make(map[string]*Client),
> 		},
> 	}
> }
> 
> // --- Client Registration ---
> 
> func (s *Server) registerClient(conn net.Conn) *Client {
> 	fmt.Fprint(conn, "Welcome to Go Chat! Enter your nickname: ")
> 	scanner := bufio.NewScanner(conn)
> 
> 	for scanner.Scan() {
> 		nick := strings.TrimSpace(scanner.Text())
> 		if nick == "" {
> 			fmt.Fprint(conn, "Nickname cannot be empty. Try again: ")
> 			continue
> 		}
> 		if strings.HasPrefix(nick, "/") {
> 			fmt.Fprint(conn, "Nickname cannot start with '/'. Try again: ")
> 			continue
> 		}
> 		if len(nick) > 20 {
> 			fmt.Fprint(conn, "Nickname too long (max 20 chars). Try again: ")
> 			continue
> 		}
> 
> 		s.mu.Lock()
> 		if _, exists := s.clients[nick]; exists {
> 			s.mu.Unlock()
> 			fmt.Fprintf(conn, "Nickname '%s' is taken. Try another: ", nick)
> 			continue
> 		}
> 
> 		client := &Client{
> 			conn:     conn,
> 			nickname: nick,
> 			room:     "general",
> 			outgoing: make(chan string, 256),
> 		}
> 		s.clients[nick] = client
> 		s.rooms["general"][nick] = client
> 		s.mu.Unlock()
> 
> 		return client
> 	}
> 	return nil
> }
> 
> func (s *Server) removeClient(client *Client) {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	delete(s.clients, client.nickname)
> 
> 	if room, ok := s.rooms[client.room]; ok {
> 		delete(room, client.nickname)
> 		if len(room) == 0 && client.room != "general" {
> 			delete(s.rooms, client.room)
> 		}
> 	}
> 
> 	close(client.outgoing)
> }
> 
> // --- Messaging ---
> 
> func (s *Server) broadcastToRoom(senderNick, room, msg string) {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 
> 	members, ok := s.rooms[room]
> 	if !ok {
> 		return
> 	}
> 
> 	formatted := fmt.Sprintf("[#%s] %s\n", room, msg)
> 	for nick, client := range members {
> 		if nick != senderNick {
> 			select {
> 			case client.outgoing <- formatted:
> 			default:
> 				log.Printf("Dropping message to slow client: %s", nick)
> 			}
> 		}
> 	}
> }
> 
> func (s *Server) privateMessage(sender *Client, targetNick, msg string) {
> 	s.mu.RLock()
> 	target, ok := s.clients[targetNick]
> 	s.mu.RUnlock()
> 
> 	if !ok {
> 		sender.outgoing <- fmt.Sprintf("User '%s' not found.\n", targetNick)
> 		return
> 	}
> 
> 	formatted := fmt.Sprintf("[PM from %s] %s\n", sender.nickname, msg)
> 	select {
> 	case target.outgoing <- formatted:
> 		sender.outgoing <- fmt.Sprintf("[PM to %s] %s\n", targetNick, msg)
> 	default:
> 		sender.outgoing <- fmt.Sprintf("Could not deliver message to %s (buffer full).\n", targetNick)
> 	}
> }
> 
> // --- Rooms ---
> 
> func (s *Server) joinRoom(client *Client, roomName string) {
> 	roomName = strings.TrimPrefix(roomName, "#")
> 	if roomName == "" {
> 		client.outgoing <- "Room name cannot be empty.\n"
> 		return
> 	}
> 
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	if client.room == roomName {
> 		client.outgoing <- fmt.Sprintf("You are already in #%s.\n", roomName)
> 		return
> 	}
> 
> 	// Leave current room
> 	if room, ok := s.rooms[client.room]; ok {
> 		delete(room, client.nickname)
> 		for _, c := range room {
> 			select {
> 			case c.outgoing <- fmt.Sprintf("[#%s] * %s left the room\n", client.room, client.nickname):
> 			default:
> 			}
> 		}
> 		if len(room) == 0 && client.room != "general" {
> 			delete(s.rooms, client.room)
> 		}
> 	}
> 
> 	// Join new room
> 	if s.rooms[roomName] == nil {
> 		s.rooms[roomName] = make(map[string]*Client)
> 	}
> 	s.rooms[roomName][client.nickname] = client
> 	oldRoom := client.room
> 	client.room = roomName
> 
> 	// Notify new room
> 	for nick, c := range s.rooms[roomName] {
> 		if nick != client.nickname {
> 			select {
> 			case c.outgoing <- fmt.Sprintf("[#%s] * %s joined the room\n", roomName, client.nickname):
> 			default:
> 			}
> 		}
> 	}
> 
> 	_ = oldRoom
> 	client.outgoing <- fmt.Sprintf("Joined #%s (%d users)\n",
> 		roomName, len(s.rooms[roomName]))
> }
> 
> func (s *Server) leaveRoom(client *Client) {
> 	if client.room == "general" {
> 		client.outgoing <- "You are already in the general room.\n"
> 		return
> 	}
> 	s.joinRoom(client, "general")
> }
> 
> func (s *Server) listRooms(client *Client) {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 
> 	var buf strings.Builder
> 	buf.WriteString("Active rooms:\n")
> 	for name, members := range s.rooms {
> 		marker := ""
> 		if name == client.room {
> 			marker = " <- you are here"
> 		}
> 		buf.WriteString(fmt.Sprintf("  #%-15s %d users%s\n", name, len(members), marker))
> 	}
> 	client.outgoing <- buf.String()
> }
> 
> func (s *Server) listUsers(client *Client) {
> 	s.mu.RLock()
> 	defer s.mu.RUnlock()
> 
> 	var buf strings.Builder
> 	buf.WriteString("Online users:\n")
> 	for nick, c := range s.clients {
> 		buf.WriteString(fmt.Sprintf("  %-20s #%s\n", nick, c.room))
> 	}
> 	buf.WriteString(fmt.Sprintf("Total: %d users\n", len(s.clients)))
> 	client.outgoing <- buf.String()
> }
> 
> // --- Command Handling ---
> 
> func (s *Server) handleCommand(sender *Client, line string) bool {
> 	parts := strings.SplitN(line, " ", 3)
> 	cmd := strings.ToLower(parts[0])
> 
> 	switch cmd {
> 	case "/msg", "/pm", "/whisper":
> 		if len(parts) < 3 {
> 			sender.outgoing <- "Usage: /msg <nickname> <message>\n"
> 			return true
> 		}
> 		s.privateMessage(sender, parts[1], parts[2])
> 		return true
> 
> 	case "/join":
> 		if len(parts) < 2 {
> 			sender.outgoing <- "Usage: /join <room>\n"
> 			return true
> 		}
> 		s.joinRoom(sender, parts[1])
> 		return true
> 
> 	case "/leave":
> 		s.leaveRoom(sender)
> 		return true
> 
> 	case "/rooms":
> 		s.listRooms(sender)
> 		return true
> 
> 	case "/who", "/users":
> 		s.listUsers(sender)
> 		return true
> 
> 	case "/nick":
> 		if len(parts) < 2 {
> 			sender.outgoing <- "Usage: /nick <new_nickname>\n"
> 			return true
> 		}
> 		s.changeNick(sender, strings.TrimSpace(parts[1]))
> 		return true
> 
> 	case "/help":
> 		sender.outgoing <- `Commands:
>   /msg <nick> <text>  Send a private message
>   /join <room>        Join a chat room
>   /leave              Return to #general
>   /rooms              List active rooms
>   /who                List online users
>   /nick <name>        Change your nickname
>   /quit               Disconnect
>   /help               Show this help
> `
> 		return true
> 
> 	case "/quit", "/exit":
> 		sender.outgoing <- "Goodbye!\n"
> 		sender.conn.Close()
> 		return true
> 	}
> 
> 	return false
> }
> 
> func (s *Server) changeNick(client *Client, newNick string) {
> 	if newNick == "" || strings.HasPrefix(newNick, "/") || len(newNick) > 20 {
> 		client.outgoing <- "Invalid nickname.\n"
> 		return
> 	}
> 
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	if _, exists := s.clients[newNick]; exists {
> 		client.outgoing <- fmt.Sprintf("Nickname '%s' is already taken.\n", newNick)
> 		return
> 	}
> 
> 	oldNick := client.nickname
> 
> 	// Update client registry
> 	delete(s.clients, oldNick)
> 	s.clients[newNick] = client
> 
> 	// Update room membership
> 	if room, ok := s.rooms[client.room]; ok {
> 		delete(room, oldNick)
> 		room[newNick] = client
> 	}
> 
> 	client.nickname = newNick
> 	client.outgoing <- fmt.Sprintf("Nickname changed to '%s'\n", newNick)
> 
> 	// Announce to room
> 	if room, ok := s.rooms[client.room]; ok {
> 		msg := fmt.Sprintf("[#%s] * %s is now known as %s\n", client.room, oldNick, newNick)
> 		for nick, c := range room {
> 			if nick != newNick {
> 				select {
> 				case c.outgoing <- msg:
> 				default:
> 				}
> 			}
> 		}
> 	}
> }
> 
> // --- Client Handler ---
> 
> func (s *Server) handleClient(client *Client) {
> 	defer s.removeClient(client)
> 
> 	// Start write pump
> 	go client.writePump()
> 
> 	// Announce join
> 	s.broadcastToRoom(client.nickname, client.room,
> 		fmt.Sprintf("* %s has connected", client.nickname))
> 	client.outgoing <- fmt.Sprintf("Welcome, %s! You are in #%s. Type /help for commands.\n",
> 		client.nickname, client.room)
> 
> 	// Read loop
> 	scanner := bufio.NewScanner(client.conn)
> 	for scanner.Scan() {
> 		line := strings.TrimSpace(scanner.Text())
> 		if line == "" {
> 			continue
> 		}
> 
> 		if strings.HasPrefix(line, "/") {
> 			if s.handleCommand(client, line) {
> 				continue
> 			}
> 			client.outgoing <- fmt.Sprintf("Unknown command: %s. Type /help for help.\n",
> 				strings.SplitN(line, " ", 2)[0])
> 			continue
> 		}
> 
> 		// Regular message: broadcast to room
> 		s.broadcastToRoom(client.nickname, client.room,
> 			fmt.Sprintf("[%s] %s", client.nickname, line))
> 	}
> 
> 	// Announce leave
> 	s.broadcastToRoom(client.nickname, client.room,
> 		fmt.Sprintf("* %s has disconnected", client.nickname))
> 	log.Printf("Client disconnected: %s", client.nickname)
> }
> 
> // --- Shutdown ---
> 
> func (s *Server) shutdown() {
> 	s.mu.Lock()
> 	defer s.mu.Unlock()
> 
> 	log.Printf("Shutting down, disconnecting %d clients...", len(s.clients))
> 	for _, client := range s.clients {
> 		fmt.Fprint(client.conn, "\nServer is shutting down. Goodbye!\n")
> 		client.conn.Close()
> 	}
> }
> 
> // --- Main ---
> 
> func main() {
> 	addr := ":9000"
> 	if len(os.Args) > 1 {
> 		addr = os.Args[1]
> 	}
> 
> 	server := NewServer()
> 
> 	listener, err := net.Listen("tcp", addr)
> 	if err != nil {
> 		log.Fatalf("Failed to start server: %v", err)
> 	}
> 
> 	// Handle shutdown signal
> 	ctx, cancel := context.WithCancel(context.Background())
> 	defer cancel()
> 
> 	sigCh := make(chan os.Signal, 1)
> 	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
> 
> 	go func() {
> 		<-sigCh
> 		log.Println("Received shutdown signal")
> 		cancel()
> 		listener.Close() // unblocks Accept()
> 	}()
> 
> 	log.Printf("Chat server listening on %s", addr)
> 	log.Println("Connect with: nc localhost 9000")
> 
> 	for {
> 		conn, err := listener.Accept()
> 		if err != nil {
> 			select {
> 			case <-ctx.Done():
> 				server.shutdown()
> 				log.Println("Server stopped.")
> 				return
> 			default:
> 				log.Printf("Accept error: %v", err)
> 				continue
> 			}
> 		}
> 
> 		go func() {
> 			client := server.registerClient(conn)
> 			if client == nil {
> 				conn.Close()
> 				return
> 			}
> 			log.Printf("Client registered: %s (%s)", client.nickname, conn.RemoteAddr())
> 			server.handleClient(client)
> 		}()
> 	}
> }
> ```
>
> **Test it:**
> ```bash
> # Terminal 1: Start the server
> go run main.go
>
> # Terminal 2: Connect as Alice
> nc localhost 9000
> # Type: Alice
> # Type: Hello everyone!
>
> # Terminal 3: Connect as Bob
> nc localhost 9000
> # Type: Bob
> # Type: Hey Alice!
> # Type: /msg Alice this is private
> # Type: /join golang
> # Type: Anyone here?
>
> # Terminal 4: Connect as Charlie
> nc localhost 9000
> # Type: Charlie
> # Type: /join golang
> # Type: Hi Bob!
> # Type: /rooms
> # Type: /who
> ```

---

## Extensions / Challenges

- **Message history:** Store the last N messages per room so new joiners see recent context.
- **Typing indicator:** When a user starts typing, notify others in the room (requires a custom protocol beyond line-by-line).
- **Kick/ban:** Add `/kick` and `/ban` commands for room moderators (the first user in a room is the moderator).
- **File sharing:** Allow sending small files encoded in base64.
- **TLS:** Add TLS support with `crypto/tls` for encrypted connections.
- **Persistence:** Save room state and message history to a file or SQLite database.
- **Client GUI:** Build a simple terminal UI client using `github.com/charmbracelet/bubbletea`.

> [!abstract] Key Takeaways
> - Each TCP connection gets its own goroutine pair: one for reading, one for writing.
> - The outgoing channel per client decouples message production from slow network writes.
> - `select` with `default` provides non-blocking sends, essential for protecting the server from slow clients.
> - `sync.RWMutex` allows concurrent reads of the client/room maps while serializing writes.
> - Closing the listener unblocks `Accept()`, enabling clean shutdown in the accept loop.
> - Always set write deadlines on connections to prevent goroutine leaks from unresponsive clients.
