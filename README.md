# newsrank-realtime-go

Official Go SDK for the [NewsRank](https://newsrank.ai) Realtime WebSocket API. Receive breaking news, story developments, and ranking changes the moment they happen.

Requires **Go 1.22+**.

## Install

```bash
go get github.com/newsrank-ai/newsrank-realtime-go
```

## Quick Start

```go
package main

import (
	"context"
	"fmt"
	"log"

	realtime "github.com/newsrank-ai/newsrank-realtime-go"
)

func main() {
	client := realtime.NewClient("nrf_your_api_key")

	client.OnItemNew(func(ctx context.Context, e *realtime.ItemNewEvent) {
		fmt.Printf("[%s] %s\n", e.Data.SourceName, e.Data.Title)
		fmt.Printf("  %s\n", e.Data.URL)
	})

	client.OnStoryDevelopment(func(ctx context.Context, e *realtime.StoryDevelopmentEvent) {
		fmt.Printf("Story update: %s\n", e.Data.Summary)
	})

	ctx := context.Background()
	if err := client.Dial(ctx); err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	err := client.Subscribe(ctx, &realtime.SubscribeFilters{
		Categories: []string{"politics", "tech"},
	})
	if err != nil {
		log.Fatal(err)
	}

	// Block and dispatch events until context is cancelled
	if err := client.Listen(ctx); err != nil {
		log.Fatal(err)
	}
}
```

## Events

Register typed handlers with `On*` methods:

```go
client.OnItemNew(func(ctx context.Context, e *realtime.ItemNewEvent) { ... })
client.OnItemEnriched(func(ctx context.Context, e *realtime.ItemEnrichedEvent) { ... })
client.OnClustersRanked(func(ctx context.Context, e *realtime.ClustersRankedEvent) { ... })
client.OnStoryDevelopment(func(ctx context.Context, e *realtime.StoryDevelopmentEvent) { ... })
client.OnStoryPriorityChanged(func(ctx context.Context, e *realtime.StoryPriorityChangedEvent) { ... })
client.OnClusterEvent(func(ctx context.Context, e *realtime.ClusterEventEvent) { ... })
client.OnSubscribed(func(ctx context.Context, e *realtime.SubscribedEvent) { ... })
client.OnError(func(ctx context.Context, e *realtime.ErrorEvent) { ... })
```

| Event Constant | Description |
|---------------|-------------|
| `EventItemNew` | New article matching your filters |
| `EventItemEnriched` | Article enriched with entities and metadata |
| `EventClustersRanked` | Updated story rankings |
| `EventStoryDevelopment` | New development in an ongoing story |
| `EventStoryPriorityChanged` | Story priority level changed |
| `EventClusterEvent` | Cluster lifecycle (created/merged/split) |
| `EventSubscribed` | Subscription confirmed |
| `EventError` | Server-side error |

## Filters

```go
client.Subscribe(ctx, &realtime.SubscribeFilters{
    Categories:       []string{"politics", "tech", "world"},
    Sources:          []string{"reuters", "ap-news"},
    Keywords:         []string{"AI", "regulation"},
    Entities:         []int{42, 108},
    Topics:           []string{"articles"},
    IncludeEmbeddings: true,                           // Pro+ only
    EmbeddingFilters: []realtime.EmbeddingFilter{{     // Pro+ only
        Embedding: []float64{0.1, 0.2},
        Threshold: 0.8,
        Label:     "my-topic",
    }},
})
```

Filters are remembered and automatically replayed on reconnect.

## Options

```go
client := realtime.NewClient("nrf_...",
    realtime.WithURL("wss://api.newsrank.ai/ws"),        // WebSocket URL (default)
    realtime.WithAutoReconnect(true),                     // Auto-reconnect (default: true)
    realtime.WithMaxReconnectAttempts(0),                  // 0 = unlimited (default)
    realtime.WithHeartbeatInterval(30 * time.Second),     // Heartbeat interval (default: 30s)
)
```

## Error Handling

```go
if err := client.Dial(ctx); err != nil {
    if errors.Is(err, realtime.ErrAuthFailed) {
        log.Fatal("Invalid API key")
    }
    log.Fatal("Connection failed:", err)
}

client.OnError(func(ctx context.Context, e *realtime.ErrorEvent) {
    log.Printf("Server error: %s", e.Message)
})
```

**Sentinel errors:**

- `ErrNotConnected` - operation attempted while disconnected
- `ErrAlreadyConnected` - `Dial()` called on an already-connected client
- `ErrAuthFailed` - invalid API key
- `ErrConnectionLimit` - connection limit reached

## API Reference

### `NewClient(apiKey string, opts ...Option) *Client`

Create a new client with functional options.

### `client.Dial(ctx) error`

Connect to the WebSocket server.

### `client.Close() error`

Gracefully close the connection.

### `client.Subscribe(ctx, filters) error`

Subscribe with optional filters. Replayed on reconnect.

### `client.Listen(ctx) error`

Block and dispatch events. Handles reconnection with exponential backoff.

### `client.Connected() bool`

Whether the WebSocket is currently connected.

## Links

- [Full Documentation](https://newsrank.ai/docs/sdks)
- [WebSocket API Reference](https://newsrank.ai/docs/websocket)
- [REST API Docs](https://newsrank.ai/docs/getting-started)
- [Get an API Key](https://newsrank.ai/dashboard/developers)

## License

MIT
