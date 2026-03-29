# Archers - a program for sending tabular data over a network using Arrow

I decided to write this as a learning project that might also be useful. It doesn't aim to replace SSH, rsync, etc. It's its own thing. 

## Stage 1: Async foundations — before touching Flight at all
- [ ] Step 1: Read about async Rust conceptually before writing any.
Understand what async fn and .await mean, what a runtime is and why you need one (unlike Go, Rust has no built-in async runtime — tokio is the standard choice), and what a Future is at a high level. You don't need to understand Pin and Waker deeply yet — just know they exist and why. The Rust async book at https://rust-lang.github.io/async-book is the right starting point.
- [ ] Step 2: Write a trivial tokio program.
Just get tokio in your Cargo.toml and write a main that spawns a task, does something async, and awaits it. The #[tokio::main] macro is your entry point. Get comfortable with the fact that async fn main() doesn't work without it.
- [ ] Step 3: Write a simple TCP echo server and client using tokio::net.
Before Flight exists in the picture, write a program that sends a string over a TCP connection and receives it back. This teaches you TcpListener, TcpStream, and the async read/write traits — the plumbing that everything else sits on. It's also where you'll first fight the borrow checker across await points, which is better to experience in a simple context.

## Stage 2: gRPC and tonic
- [ ] Step 4: Understand why Flight is built on gRPC.
Arrow Flight uses gRPC as its transport. tonic is Rust's gRPC implementation. Read enough about gRPC to understand the request/response and streaming models — particularly server streaming (one request, many responses) which is the Flight pattern for sending a large dataset.
- [ ] Step 5: Write a trivial tonic gRPC service from scratch.
Before touching Flight, define a minimal .proto file yourself — something like a service that takes a name and returns a greeting. Use tonic-build in a build.rs to generate Rust code from it. Get a server and client talking. This teaches you the tonic pattern and the build.rs concept, which is Rust's mechanism for code generation at compile time.

## Stage 3: Arrow Flight specifically
- [ ] Step 6: Add arrow-flight to your dependencies and read the crate docs.
Look at the types FlightServiceServer, FlightServiceClient, and the FlightData message. Notice that FlightData is essentially a wrapper around raw Arrow IPC bytes — that's why it fits your use case so naturally.
- [ ] Step 7: Implement a minimal Flight server.
A Flight server implements the FlightService trait, which has several methods — don't try to implement all of them at once. Start with just do_get, which is the server-streaming RPC that sends a stream of FlightData to a client. Hardcode a small RecordBatch to send for now — don't worry about reading from files yet.
- [ ]Step 8: Implement the client side.
Write the receive subcommand that connects to the server, calls do_get, and receives the stream of FlightData back. Deserialise it into RecordBatches and print the row count to confirm it worked. At this point you have a working end-to-end Flight connection.

## Stage 4: Connecting it to your existing work
- [ ] Step 9: Add clap for subcommand argument parsing.
Structure the binary so mytool send <file> --host <addr> and mytool receive --host <addr> <output.parquet> work as subcommands. clap's derive API lets you define your CLI as a struct with annotations, which is very clean and worth learning over the builder API for a project this size.
- [ ] Step 10: Wire the sender to your existing CSV/Parquet pipeline.
The send path should read from a file (reusing your MultiDelimReader and zip logic if needed), produce RecordBatches, and stream them as FlightData to the server. This is where the zero-copy story comes together — a RecordBatch can be IPC-encoded directly into the FlightData wire format without an intermediate copy if you use the Arrow IPC writer correctly.
- [ ] Step 11: Wire the receiver to write Parquet.
The receive path takes the incoming FlightData stream, deserialises it back to RecordBatches, and writes them to a Parquet file using your existing flush_batch logic. At this point the two tools are essentially a pipeline.

## Stage 5: Polish
- [ ] Step 12: Add authentication and TLS.
tonic has built-in support for both. This is optional for a research tool but worth knowing about, and it's where you'd need it if this ever crossed a network you don't control.
Step 13: Benchmark and think about backpressure.
With async streaming, the sender can produce data faster than the receiver can consume it. tokio and tonic handle backpressure automatically through their channel mechanics, but understanding why it works is worth the time — it's one of the things async Rust gets genuinely right.
