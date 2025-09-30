# Chapter 13: TCP Connection Management

## Miscellaneous Notes

### TCP TIME_WAIT Socket Reuse

- `sysctl -w net.ipv4.tcp_tw_reuse=1` can be used to reuse sockets (and ports) taken up by TIME_WAIT connections
- This addresses the TIME_WAIT issue that can occur with high-traffic applications

### Connection Pooling vs Socket Reuse

- A better solution than socket reuse is to reuse TCP connections by returning them to the HTTP client's connection pool
- Some libraries (like Iceberg) close connections if all bytes aren't read from the stream
- HTTP clients typically read remaining bytes to clear the socket buffer before reusing connections