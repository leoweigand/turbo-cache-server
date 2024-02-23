<p align="center"><br><img src="./icon.png" width="128" height="128" alt="Turbo engine" /></p>
<h2 align="center">Turbo Cache Server</h2>
<p align="center">
  <a href="https://turbo.build/repo">Turborepo</a> remote cache server as a Github Action with S3-compatible storage support.
</p>

### How can I use this in my monorepo?

You can use the Turbo Cache Server as **Github Action**. Here is how:

1. On your workflow files, add the following global environment variables:

```yml
env:
  TURBO_API: "http://127.0.0.1:8585"
  TURBO_TEAM: "NAME_OF_YOUR_REPO_HERE"
  # The value of TURBO_TOKEN is irrelevant
  # as we don't perform any auth against the cache server
  # but it's still necessary for Turborepo
  TURBO_TOKEN: "turbo-token"
```

> [!NOTE]
> These environment variables are required by Turborepo so it can call
> the Turbo Cache Server with the right HTTP body, headers and query strings.
> These environment variables are necessary so the Turborepo binary can
> identify the Remote Cache feature is enabled and can use them across all steps.
> You can [read more about this here](https://turbo.build/repo/docs/ci#setup) on the Turborepo official docs.

Make sure that you have an S3-compatible storage available. We currently tested with:

- [Amazon S3](https://aws.amazon.com/s3/)
- [Cloudflare R2](https://www.cloudflare.com/en-gb/developer-platform/r2/)
- [Minio Object Storage](https://min.io/)

1. Still on your `yml` file, after checking out your code, use our custom
   action to start the Turbo Cache Server in the background:

```yml
- name: Checkout repository
  uses: actions/checkout@v4

- name: Turborepo Cache Server
  uses: brunojppb/turbo-cache-server@1.0.0
  env:
    PORT: "8585"
    S3_ACCESS_KEY: "YOUR_S3_ACCESS_KEY"
    S3_SECRET_KEY: "YOUR_S3_SECRET_KEY"
    S3_ENDPOINT": "YOUR_S3_ENDPOINT"
    S3_BUCKET_NAME: "YOUR_BUCKET_NAME"
    # Region defaults to "eu-central-1"
    S3_REGION: "eu-central-1"
    # if your S3-compatible store does not support requests
    # like https://bucket.hostname.domain/. Setting `S3_USE_PATH_STYLE`
    # to true configures the S3 client to make requests like
    # https://hostname.domain/bucket instead.
    # Defaults to "false"
    S3_USE_PATH_STYLE: false
```

And that is all you need to use our remote cache server for Turborepo.

## How does that work?

Turbo Cache Server is a tiny web server written in [Rust](https://www.rust-lang.org/) that
uses any S3-compatible bucket as its storage layer for the artifacts generated by Turborepo.

### What happens when there is a cache hit?

Here is a diagram showing how the Turbo Cache Server works within our actions during a cache hit:

```mermaid
sequenceDiagram
    actor A as Developer
    participant B as Github
    participant C as Github Actions
    participant D as Turbo Cache Server
    participant E as S3 bucket
    A->>+B: Push new commit to GH.<br>Trigger PR Checks.
    B->>+C: Trigger CI pipeline
    C->>+D: turborepo cache server via<br/>"use: turbo-cache-server@0.0.2" action
    Note right of C: Starts a server instance<br/> in the background.
    D-->>-C: Turbo cache server ready
    C->>+D: Turborepo executes task<br/>(e.g. test, build)
    Note right of C: Cache check on the Turbo cache server<br/>for task hash "1wa2dr3"
    D->>+E: Get object with name "1wa2dr3"
    E-->>-D: object "1wa2dr3" exists
    D-->>-C: Cache hit for task "1wa2dr3"
    Note right of C: Replay logs and artifacts<br/>for task
    C->>+D: Post-action: Shutdown Turbo Cache Server
    D-->>-C: Turbo Cache server terminates safely
    C-->>-B: CI pipline complete
    B-->>-A: PR Checks done
```

### What happens when there is a cache miss?

When a cache isn't yet available, the Turbo Cache Server will handle new uploads and store the
artifacts in S3 as you can see in the following diagram:

```mermaid
sequenceDiagram
    actor A as Developer
    participant B as Github
    participant C as Github Actions
    participant D as Turbo Cache Server
    participant E as S3 bucket
    A->>+B: Push new commit to GH.<br>Trigger PR Checks.
    B->>+C: Trigger CI pipeline
    C->>+D: turborepo cache server via<br/>"use: turbo-cache-server@0.0.2" action
    Note right of C: Starts a server instance<br/> in the background.
    D-->>-C: Turborepo cache server ready
    C->>+D: Turborepo executes build task
    Note right of C: Cache check on the server<br/>for task hash "1wa2dr3"
    D->>+E: Get object with name "1wa2dr3"
    E-->>-D: object "1wa2dr3" DOES NOT exist
    D-->>-C: Cache miss for task "1wa2dr3"
    Note right of C: Turborepo executes task normaly
    C-->>C: Turborepo executes build task
    C->>+D: Turborepo uploads cache artifact<br/>with hash "1wa2dr3"
    D->>+E: Put object with name "1wa2dr3"
    E->>-D: Object stored
    D-->>-C: Cache upload complete
    C->>+D: Post-action: Turbo Cache Server shutdown
    D-->>-C: Turbo Cache server terminates safely
    C-->>-B: CI pipline complete
    B-->>-A: PR Checks done
```

## Development

Turbo Cache Server requires [Rust](https://www.rust-lang.org/) 1.75 or above. To setup your
environment, use the rustup script as recommended by the
[Rust docs](https://www.rust-lang.org/learn/get-started):

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Now run the following command to run the web server locally:

```shell
cargo run
```

### Setting up your environment

During local development, you might want to try the Turbo Dev Server locally against a JS monorepo. As it depends on a S3-compatible service for storing Turborepo artifacts, we recommend using [Minio](https://min.io/) with Docker with the following command:

```shell
docker run \
  -d \
  -p 9000:9000 \
  -p 9001:9001 \
  --user $(id -u):$(id -g) \
  --name minio1 \
  -e "MINIO_ROOT_USER=minio" \
  -e "MINIO_ROOT_PASSWORD=minio12345" \
  -v ./s3_data:/data \
  quay.io/minio/minio server /data --console-address ":9001"
```

#### Setting up environment variables

Copy the `.env.example` file, rename it to `.env` and add the environment
variables required. As we use Minio locally, just go to the
[Web UI](http://localhost:9001) of Minio, create a bucket and generate
credentials and copy it to the `.env` file.

### Tests

To execute the test suite, run:

```shell
cargo test
```
