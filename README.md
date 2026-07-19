# Internet-Banking-Project
This Source code works single entry point for all client traffic, handling JWT auth, idempotency enforcement, rate limiting (via Redis), routing, and circuit breaking to downstream microservices (ledger, payment orchestration, UPI, international payments, insurance, KYC).
