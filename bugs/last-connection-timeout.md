# Last Connection Timeout Issue

## Summary
The last QUIC connection in a sequential batch experiences stream failures with `ConnectionLost(TimedOut)` errors, while all previous connections complete successfully.

## Observed Behavior
- Connections 1-98: Complete successfully (all 20,000 streams processed)
- Connection 99: Only ~57% of streams complete (10,365 passes, 1,140 fails)
- Error: `ConnectionLost(TimedOut)` on failed streams

## Root Cause
The client processes connections sequentially using `.await` between each connection. When the final connection finishes sending streams, `main()` returns immediately and the program exits. This abruptly terminates the connection before the server finishes reading all streams.

### Technical Details
- `send_stream.finish()` (client.rs:52) only queues the FIN packet, doesn't wait for acknowledgment
- The server task (spawned at server.rs:49) races to read streams before client shutdown
- Early connections have buffer time while subsequent connections are being processed
- The last connection has no buffer time before program termination

## Fix
Add a delay before program exit to allow the last connection to drain:

```rust
// After the connection loop in send_streams()
tokio::time::sleep(Duration::from_secs(5)).await;
```

## Alternative Solutions
1. Wait for graceful connection closure: `conn.close(0u32.into(), b"done"); conn.closed().await;`
2. Keep the program running: Add infinite sleep or wait for user input
3. Process connections in parallel instead of sequentially (would need coordination for clean exit)

## Date
2026-01-05
