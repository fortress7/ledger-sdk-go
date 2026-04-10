# Ledger Go SDK

Go client SDK for the Ledger financial accounting engine.

## Installation

```bash
go get github.com/fortress7/ledger-sdk-go
```

## Quick Start

```go
package main

import (
	"context"
	"fmt"
	"log"

	ledger "github.com/fortress7/ledger-sdk-go/gen/ledger"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	conn, err := grpc.NewClient("localhost:45000",
		grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	// Create a ledger
	ledgerClient := ledger.NewLedgerDataApplicationServiceClient(conn)
	resp, err := ledgerClient.CreateLedger(context.Background(), &ledger.CreateLedgerRequest{
		LedgerName: "my_ledger",
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Ledger created: %s\n", resp.GetLedgerName())
}
```

## Available Services

The SDK provides four gRPC service clients:

### LedgerDataApplicationService

Ledger-level operations — creation, cross-account queries, audit trail.

```go
client := ledger.NewLedgerDataApplicationServiceClient(conn)

// Create a ledger
client.CreateLedger(ctx, &ledger.CreateLedgerRequest{LedgerName: "my_ledger"})

// Fetch a ledger
client.FetchLedger(ctx, &ledger.FetchLedgerRequest{LedgerName: "my_ledger"})

// List all ledgers
client.ListLedgers(ctx, &ledger.ListLedgerRequest{PageSize: 20, PageNumber: 1})

// List accounts across the ledger
client.ListAccounts(ctx, &ledger.ListAccountsRequest{...})

// Query immutable audit trail
client.ListImmutableLogs(ctx, &ledger.ListImmutableLogsRequest{...})
```

### TransactionApplicationService

Double-entry transaction recording, reversal, and querying.

```go
client := ledger.NewTransactionApplicationServiceClient(conn)

// Record a transaction
client.RecordTransaction(ctx, &ledger.RecordTransactionRequest{
	LedgerName:           "my_ledger",
	TransactionReference: "txn-001",
	Postings: []*ledger.Posting{
		{Source: "@world", Destination: "@alice", AssetCode: "USD", Amount: 100.00},
	},
})

// Record synchronously (waits for projections to complete)
client.RecordTransactionSync(ctx, &ledger.RecordTransactionRequest{...})

// Get transaction details
client.GetTransaction(ctx, &ledger.GetTransactionRequest{
	LedgerName:    "my_ledger",
	TransactionId: "txn-id",
})

// Revert a transaction
client.RevertTransaction(ctx, &ledger.RevertTransactionRequest{
	LedgerName:    "my_ledger",
	TransactionId: "original-txn-id",
})
```

### AccountApplicationService

Account and asset management.

```go
client := ledger.NewAccountApplicationServiceClient(conn)

// Get account with balances
client.GetAccount(ctx, &ledger.GetAccountRequest{
	LedgerName:     "my_ledger",
	AccountAddress: "@alice",
})

// Register an asset
client.RegisterAsset(ctx, &ledger.RegisterAssetRequest{
	LedgerName:     "my_ledger",
	AssetCode:      "USD",
	Name:           "US Dollar",
	AssetPrecision: 100,
})

// Add metadata to an account
client.AddMetadata(ctx, &ledger.AddAccountMetadataRequest{
	LedgerName:     "my_ledger",
	AccountAddress: "@alice",
	Metadata:       map[string]*ledger.AttributeValue{...},
})
```

### ReservationApplicationService

Hold/capture/release pattern for pending financial operations.

```go
client := ledger.NewReservationApplicationServiceClient(conn)

// Create a hold
client.CreateReservation(ctx, &ledger.CreateReservationRequest{
	LedgerName:     "my_ledger",
	AccountAddress: "@alice",
	AssetCode:      "USD",
	Amount:         50.00,
})

// Capture (settle) the hold
client.CaptureReservation(ctx, &ledger.CaptureReservationRequest{
	LedgerName:    "my_ledger",
	ReservationId: "reservation-id",
	Amount:        50.00,
	Destination:   "@merchant",
})

// Release (cancel) the hold
client.ReleaseReservation(ctx, &ledger.ReleaseReservationRequest{
	LedgerName:    "my_ledger",
	ReservationId: "reservation-id",
})
```

## Key Concepts

- **Ledger** — a tenant. Each ledger is isolated with its own accounts, transactions, and balances.
- **Account** — identified by an address starting with `@` (e.g., `@alice`, `@revenue`). Auto-created on first use.
- **`@world`** — special account representing the external world. Has unlimited debit capacity.
- **Transaction** — double-entry: sum of debits must equal sum of credits per asset.
- **Reservation** — holds funds on an account. Can be captured (settled) or released (cancelled).
- **Idempotency** — all write operations accept a `transactionReference` or `idempotencyKey`. Duplicate requests return the original result.

