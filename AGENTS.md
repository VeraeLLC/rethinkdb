# RethinkDB Project Overview

RethinkDB is an open-source, scalable NoSQL database designed for building realtime web applications. It stores schemaless JSON documents and supports distributed deployments with automatic failover and high availability. The database pushes updated query results to applications in realtime without polling, enabling developers to build realtime apps efficiently.

The project is primarily written in C++ and uses a custom GNU Make-based build system. It supports multiple client drivers in languages like JavaScript, Python, Ruby, Java, and others (drivers are maintained in separate repositories).

## Build and Test Commands

### Building RethinkDB

RethinkDB uses a custom build system based on GNU Make with a `./configure` script for configuration.

#### Dependencies

Required dependencies:
- GCC (>= 4.7.4) or Clang compiler
- Protocol Buffers (protobuf-compiler, libprotobuf-dev)
- jemalloc (libjemalloc-dev) - default allocator on Linux
- Ncurses (libncurses5-dev)
- Python 2.6+ or Python 3
- libcurl (libcurl4-openssl-dev)
- OpenSSL (libssl-dev, libcrypto-dev)
- wget, m4, make

Optional dependencies:
- ccache (for faster rebuilds)
- boost (system)
- gtest (for unit tests)
- re2 (regular expression library)

On Ubuntu/Debian:
```bash
sudo apt-get install build-essential protobuf-compiler python3 python-is-python3 \
    libprotobuf-dev libcurl4-openssl-dev libncurses5-dev libjemalloc-dev \
    wget m4 g++ libssl-dev
```

#### Build Steps

1. Configure the build system:
   ```bash
   ./configure --allow-fetch
   ```
   - Use `--allow-fetch` to automatically download and build missing dependencies (QuickJS is always fetched).
   - Use `--ccache` to enable ccache for faster rebuilds.
   - Use `CXX=clang++` to build with Clang instead of GCC.
   - Use `--static` to link libraries statically.

2. Build the project:
   ```bash
   make -j4
   ```
   - Use `-j<n>` for parallel builds (recommended: number of cores + 1).
   - For debug builds: `make DEBUG=1` or `make UNIT_TESTS=1`
   - To fail on warnings: `make ALLOW_WARNINGS=0`
   - For verbose output: `make VERBOSE=1`
   - To disable build countdown: `make SHOW_COUNTDOWN=0`

3. Install (optional):
   ```bash
   sudo make install
   ```

#### Platform-Specific Builds

- **Windows**: Requires Visual Studio 2022, Cygwin, Strawberry Perl, and Premake5. Run `./configure` and `make -j$(nproc)` from Cygwin shell. See `WINDOWS.md` for detailed instructions.
- **FreeBSD**: Use `gmake` instead of `make`, install dependencies with `pkg install`, and configure with `CXX=g++`.
- **OS X**: Automated via `make build-osx`. Requires macOS 10.9+.

### Testing

RethinkDB has unit tests and integration tests.

- **Unit Tests**: Located in `src/unittest/`. Build and run with:
  ```bash
  make unit
  ```
  - Use `UNIT_TEST_FILTER=<pattern>` to run specific tests.

- **Integration Tests**: Use the `test/run` script in the `test/` directory:
  ```bash
  make test
  ```
  - The `TEST` variable determines which tests to run.
  - Tests include scenarios (starting RethinkDB clusters), workloads (memcached/RDB protocol queries), and interface tests.
  - Full test lists for nightly runs are in `test/full_test/`.
  - Individual tests can be run manually:
    ```bash
    cd test && (rm -rf scratch; mkdir scratch; cd scratch; \
        ../scenarios/static_cluster.py '../memcached_workloads/append_prepend.py $HOST:$PORT')
    ```

## Code Style Guidelines

Follow the C++ coding style outlined in `STYLE.md`. Key guidelines:

- Write code in a manner similar to surrounding code.
- Use braces: `if (...) {`, `while (...) {`, `switch (...) {`, not `if(...){`.
- Header files should be included in the following order:
  1. Parent .hpp file (if you are a .cc file)
  2. C system headers (`<math.h>`, `<unistd.h>`, etc.)
  3. Standard C++ headers (`<vector>`, `<algorithm>`, etc.)
  4. Boost headers, with `"errors.hpp"` included before them
  5. Local headers with full paths (`"rdb_protocol/protocol.hpp"`, etc.)
- Avoid non-const lvalue references except in return values (use pointers instead).
- Do not use reference-type fields or turn references into pointers unexpectedly.
- Use `DISABLE_COPYING` macro for non-copyable types.
- Run `../scripts/check_style.sh` from `src/` to check for style errors.
- Suppress false positives with `// NOLINT(specific/category)`.

## Testing Instructions

### Running Tests

- **Unit tests**: `make unit` - Tests individual components in isolation.
- **Full test suite**: `make test` - Runs unit tests, ReQL tests, and integration tests.
- **Custom build directory**: Use `-b <build_dir>` with `test/run`.

### Test Types

- **Memcached Workloads** (`test/memcached_workloads/`): Scripts that run memcached protocol queries against RethinkDB servers. Workloads can be continuous (run until SIGINT) or discrete (stop on their own).
- **RDB Protocol Workloads** (`test/rdb_workloads/`): Similar to memcached workloads but use the RethinkDB ReQL protocol.
- **Scenarios** (`test/scenarios/`): Scripts that start RethinkDB clusters and run workloads against them (e.g., `static_cluster.py`, `restart.py`, `more_or_less_secondaries.py`).
- **Interface Tests** (`test/interface/`): Test server interfaces like logging or admin tables.
- **Nightly Tests** (`test/full_test/`): Test lists for automated nightly runs.

### Debugging Tests

- Run tests in isolated directories to avoid conflicts:
  ```bash
  cd test && (rm -rf scratch; mkdir scratch; cd scratch; ../scenarios/<scenario> <args>)
  ```
- Temporary files and database files are left in the scratch directory for debugging.

### Test Support Modules

The `test/common/` directory contains Python modules for writing tests:
- `driver.py`: Facilities for starting RethinkDB clusters, connecting them, and shutting them down.
- `http_admin.py`: Python wrapper around RethinkDB's HTTP interface.
- `resunder.py`: Daemon for simulating network partitions (netsplits).

## Security Considerations

- RethinkDB uses OpenSSL for TLS connections between clients and servers, and between cluster nodes.
- Supports certificate chains for TLS configuration.
- The server has an HTTP admin interface; ensure proper access controls and authentication.
- Data is stored in B-trees with log-structured serialization for durability.
- Clustering uses the Raft consensus algorithm for metadata management.
- User authentication and permissions are supported (since version 1.0 of the protocol).
- Avoid running untrusted workloads or exposing admin interfaces publicly without authentication.

## Project Structure and Architecture

### Technology Stack

- **Language**: C++ (C++11 standard)
- **Build System**: Custom GNU Make-based system (`mk/` directory) with autotools-style `./configure` script
- **Networking**: Custom RPC layer with mailboxes and directory for service discovery
- **Storage**: Custom B-tree implementation with page cache and log-structured serializer
- **Concurrency**: Cooperatively scheduled coroutines, event loops, thread pools
- **Serialization**: Protocol Buffers for wire format; custom C++ template-based serialization for internal use
- **Key Dependencies**: jemalloc, ncurses, curl, OpenSSL, protobuf, QuickJS (JavaScript execution for ReQL)

### Runtime Architecture

- **Thread Pool** (`arch/runtime/thread_pool.hpp`): Fixed number of threads running event loops. Each thread processes IO notifications and messages from other threads.
- **Blocker Pool** (`arch/io/blocker_pool.hpp`): Separate thread pool for blocking operations.
- **Coroutines** (`arch/runtime/coroutines.hpp`): Cooperatively scheduled for concurrency within threads. Can switch between threads by scheduling themselves to be resumed on another thread.
- **Networking**: TCP connections between cluster nodes, mailboxes for point-to-point communication, directory for service discovery.
- **Query Execution Flow**: Client drivers → `rdb_query_server_t` → `ql::compile_term()` → term tree evaluation → `table_query_client_t` → `primary_query_server_t` → `store_t` → B-tree operations → response.
- **Clustering**: Raft consensus algorithm for table metadata; contracts for data placement and replication.
- **Changefeeds**: Push-based realtime updates from shards to clients (handled differently from normal reads).

### Code Organization

#### Core Components

- `src/arch/`: Runtime primitives (timers, IO, coroutines, threading)
  - `runtime/`: Thread pool, coroutines, context switching, message hub
  - `io/`: Disk IO, network, blocker pool, event watcher
- `src/concurrency/`: Concurrency primitives (semaphores, mutexes, locks, signals, watchables)
- `src/rpc/`: Networking and RPC
  - `connectivity/`: Low-level TCP cluster connections
  - `mailbox/`: Point-to-point message passing
  - `directory/`: Service discovery and broadcast
  - `semilattice/`: Eventually consistent metadata storage
- `src/containers/`: Data structures and serialization (`archive/`)

#### Database Components

- `src/rdb_protocol/`: Query language (ReQL) implementation
  - `terms/`: Individual ReQL term implementations (array, object, string, math, etc.)
  - `datum.hpp/cc`: ReQL value types
  - `changefeed.hpp/cc`: Realtime push notifications
  - `store.hpp/cc`: B-tree store operations
  - `ql2.proto`: Protocol Buffer definition for client protocol
- `src/btree/`: B-tree operations (nodes, traversal, backfill)
- `src/buffer_cache/`: Page cache (`page_cache.hpp`)
- `src/serializer/`: Disk serialization (log-structured)
  - `log/`: Log-structured serializer implementation

#### Clustering Components

- `src/clustering/administration/`: Server management, command line, HTTP interface
- `src/clustering/generic/`: Raft consensus implementation
- `src/clustering/immediate_consistency/`: Query routing and consistency
- `src/clustering/query_routing/`: Table query client and primary query server
- `src/clustering/table_manager/`: Per-table management
- `src/clustering/table_contract/`: Contract-based data placement (coordinator and executor)

#### Supporting Components

- `src/unittest/`: Unit tests
- `src/extproc/`: External process execution (for JavaScript)
- `src/crypto/`: Cryptographic utilities
- `src/http/`: HTTP server for admin interface
- `src/perfmon/`: Performance monitoring
- `src/pprint/`: Pretty-printing utilities
- `src/gen/`: Generated files (web assets, etc.)

#### Tests and Scripts

- `test/`: Integration tests (scenarios, workloads, interface tests)
- `scripts/`: Utility scripts (linting, packaging, visualization)
  - `check_style.sh`: C++ style checker using cpplint
  - `rethinkdb-gdb.py`: GDB debugging helpers
- `packaging/`: Platform-specific packaging (Debian, OS X, AMI)
- `mk/`: Build system files
  - `support/pkg/`: Dependency fetching and building scripts

### Key Entry Points

- `src/main.cc`: Main entry point, delegates to subcommands
- `src/clustering/administration/main/command_line.cc`: Command-line processing
- `src/clustering/administration/main/serve.cc`: `do_serve()` - Sets up server components
- `src/rdb_protocol/query_server.cc`: Client connection handling
- `src/rdb_protocol/term.cc`: `ql::compile_term()` - ReQL compilation

## Development Conventions

- Contributions via GitHub PRs, require Developer Certificate of Origin (DCO) sign-off.
- Code reviews and CI checks are required.
- Nightly automated testing runs the full test suite.
- Style checking with `scripts/check_style.sh`.
- Web UI assets are built separately from the `old_admin` branch.
- Generated files (like web assets) are committed to `src/gen/`.

### Build Variables

Key variables in `mk/defaults.mk`:
- `DEBUG=1`: Debug build
- `UNIT_TESTS=1`: Include unit tests
- `SYMBOLS=1`: Include debug symbols
- `ALLOW_WARNINGS=1`: Do not fail on compiler warnings
- `VALGRIND=1`: Valgrind-aware build
- `COVERAGE=1`: Enable code coverage
- `STATIC=1`: Static linking

## Deployment

- **Binary installation**: `make install` (default prefix: `/usr/local`)
- **Packaging**:
  - Debian: `make build-deb-src UBUNTU_RELEASE=<name> PACKAGE_BUILD_NUMBER=<n>`
  - RPM: `scripts/build-rpm.sh`
  - OS X: `make build-osx` (produces .dmg)
  - AMI: Scripts in `packaging/ami/`
- **Clustering**: Start multiple instances; they auto-discover via networking (use `--join` to specify initial peers).
- **Configuration**: Command-line options, system tables (`rethinkdb.table_config`, `rethinkdb.db_config`) for runtime config.

## Client Protocol

The client protocol is defined in `src/rdb_protocol/ql2.proto`. Key aspects:
- Protocol Buffers-based wire format
- Handshake: Version magic number → auth key → protocol selection
- Query types: START, CONTINUE, STOP, NOREPLY_WAIT, SERVER_INFO
- Response types: SUCCESS, SUCCESS_PARTIAL, WAIT_COMPLETE, CURSOR_EMPTY, CLIENT_ERROR, COMPILE_ERROR, RUNTIME_ERROR

## Changefeeds

Changefeeds provide realtime push notifications:
- Client subscribes by sending a `read_t` with a mailbox address for `changefeed::client_t`
- Shards subscribe the client to their `changefeed::server_t` (member of `store_t`)
- Data flows from shards to clients without polling
- Implementation in `src/rdb_protocol/changefeed.hpp`

## Table Configuration and Raft

- Each table has a Raft cluster for metadata consensus
- `table_raft_state_t` contains configuration and contracts
- `contract_t` objects describe what each server should do for each region
- `contract_coordinator_t` (on Raft leader) updates contracts to match config
- `contract_executor_t` (on all servers) constructs `store_t`, `primary_query_server_t`, etc. based on contracts
- See `clustering/table_contract/contract_metadata.hpp` for detailed comments
