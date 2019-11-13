# go-docker
simple golang app for docker
Referenced from here: https://www.callicoder.com/docker-golang-image-container-example/

Creating a Simple Golang App
Let’s create a simple Go app that we’ll containerize. Fire up your terminal and type the following command to create a Go project -

$ mkdir go-docker
We’ll use Go modules for dependency management. Change to the root directory of the project and initialize Go modules like so -

$ cd go-docker
$ go mod init github.com/callicoder/go-docker
We’ll be creating a simple Hello world server. Create a new file called hello_server.go -

$ touch hello_server.go
Following are the contents of the hello_server.go file -

package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gorilla/mux"
	"gopkg.in/natefinch/lumberjack.v2"
)

func handler(w http.ResponseWriter, r *http.Request) {
	query := r.URL.Query()
	name := query.Get("name")
	if name == "" {
		name = "Guest"
	}
	log.Printf("Received request for %s\n", name)
	w.Write([]byte(fmt.Sprintf("Hello, %s\n", name)))
}

func main() {
	// Create Server and Route Handlers
	r := mux.NewRouter()

	r.HandleFunc("/", handler)

	srv := &http.Server{
		Handler:      r,
		Addr:         ":8080",
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Configure Logging
	LOG_FILE_LOCATION := os.Getenv("LOG_FILE_LOCATION")
	if LOG_FILE_LOCATION != "" {
		log.SetOutput(&lumberjack.Logger{
			Filename:   LOG_FILE_LOCATION,
			MaxSize:    500, // megabytes
			MaxBackups: 3,
			MaxAge:     28,   //days
			Compress:   true, // disabled by default
		})
	}

	// Start Server
	go func() {
		log.Println("Starting Server")
		if err := srv.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	// Graceful Shutdown
	waitForShutdown(srv)
}

func waitForShutdown(srv *http.Server) {
	interruptChan := make(chan os.Signal, 1)
	signal.Notify(interruptChan, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

	// Block until we receive our signal.
	<-interruptChan

	// Create a deadline to wait for.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	srv.Shutdown(ctx)

	log.Println("Shutting down")
	os.Exit(0)
}
The server uses gorilla mux to create HTTP routes. It listens for connections on port 8080.

Building and Running the app locally
Let’s first build and run our application locally. Please type the following command to build the app -

$ go build
The build command will produce an executable file named go-docker. You can run the binary executable like so -

$ ./go-docker
2018/12/22 19:16:02 Starting Server
Our hello server is now running. Try interacting with the hello server using curl -

$ curl http://localhost:8080
Hello, Guest

$ curl http://localhost:8080?name=Rajeev
Hello, Rajeev
Defining the Docker image using a Dockerfile
Let’s define the Docker image for our Go application. Create a new file called Dockerfile inside the root directory of your project with the following contents -

# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from the latest golang base image
FROM golang:latest

# Add Maintainer Info
LABEL maintainer="Rajeev Singh <rajeevhub@gmail.com>"

# Set the Current Working Directory inside the container
WORKDIR /app

# Copy go mod and sum files
COPY go.mod go.sum ./

# Download all dependencies. Dependencies will be cached if the go.mod and go.sum files are not changed
RUN go mod download

# Copy the source from the current directory to the Working Directory inside the container
COPY . .

# Build the Go app
RUN go build -o main .

# Expose port 8080 to the outside world
EXPOSE 8080

# Command to run the executable
CMD ["./main"]
