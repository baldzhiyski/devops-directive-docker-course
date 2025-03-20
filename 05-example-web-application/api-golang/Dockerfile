# Pin specific version for stability
# Use a multi-stage build to optimize the Docker image.
# The first stage is responsible for compiling dependencies and building the application.
# The final stage will use the built binaries and create a much smaller image.
# `golang:1.19-bullseye` is used as the base image for the build because it comes with Go preinstalled.

# Stage 1: Build Base Image for Dependencies
FROM golang:1.19-bullseye AS build-base

# Set the working directory inside the container
WORKDIR /app

# Copy go.mod and go.sum first to ensure caching of dependency downloads
# This is more efficient as it allows Docker to cache these files, avoiding downloading dependencies on each build.
COPY go.mod go.sum ./

# Use cache mounts to speed up the dependency installation by storing downloaded dependencies and Go build cache
# These cache mounts will persist across builds to avoid re-downloading dependencies every time.
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go mod download  # Download all dependencies defined in go.mod

# Stage 2: Development Image
FROM build-base AS dev

# Install development tools:
# - `air` for hot-reloading the application during development
# - `dlv` for debugging the Go application
RUN go install github.com/cosmtrek/air@latest && \
    go install github.com/go-delve/delve/cmd/dlv@latest

# Copy the full project files into the container
# This will include your application code.
COPY . ./

# Start the application using `air` with the provided `.air.toml` configuration for live reloading
CMD ["air", "-c", ".air.toml"]

# Stage 3: Build Production Image
FROM build-base AS build-production

# Create a non-root user to ensure security when running the application in production
RUN useradd -u 1001 nonroot

# Copy the full project files into the container for building the application
COPY . ./

# Compile a `healthcheck` binary which will be used to check the health of the container
RUN go build \
    -ldflags="-linkmode external -extldflags -static" \  
    -tags netgo \  
    -o healthcheck \  
    ./healthcheck/healthcheck.go

# Compile the main application binary (`api-golang`) with similar flags
# The `-tags netgo` ensures Go networking is statically linked to avoid relying on external network libraries.
RUN go build \
    -ldflags="-linkmode external -extldflags -static" \  
    -tags netgo \ 
    -o api-golang  

# Stage 4: Production Deployable Image
FROM scratch

# Set the Gin framework's mode to "release" for optimized performance
ENV GIN_MODE=release

# Set the working directory in the final deployable container
WORKDIR /

# Copy the passwd file to allow the `nonroot` user to work in the container
COPY --from=build-production /etc/passwd /etc/passwd

# Copy the `healthcheck` binary compiled in the build stage into the final container
COPY --from=build-production /app/healthcheck/healthcheck healthcheck

# Copy the main application binary (`api-golang`) into the final container
COPY --from=build-production /app/api-golang api-golang

# Switch to using the `nonroot` user for running the application to enhance security
USER nonroot

# Expose port 8080 which is the default for the application
EXPOSE 8080

# Set the default command to run the application (`api-golang`)
CMD ["/api-golang"]
