Medium Severity Findings
futarchy_core
Request ID Collision Risk - resource_requests.move uses UIDs for tracking, but resource receipts with dummy ID (@0x0) could collide if compared
Sponsorship Auth Expiry Check - sponsorship_auth.move validates expiry but could be clearer about edge cases
futarchy_proposal
Lifecycle State Machine - Design consideration around state transitions
futarchy_factory
Factory Initialization - Multiple initialization paths could lead to inconsistent state
Launchpad Configuration - Fee/quota validation timing
DAO Creation - Asset type validation
futarchy_governance_actions
Public GovernanceWitness Constructor - governance_intents.move:34-36 allows anyone to obtain a witness
Scan Position in remove_from_index - intent_janitor.move:337-367 edge case with swap_remove
futarchy_markets_core
MustShare Has drop Ability - unified_spot_pool.move:110 - Comment says "no abilities" but has drop
futarchy_markets_primitives
Public drop_*_progress Functions - coin_escrow.move:1471-1533 documented as package-only
Inconsistent Quantum Invariant Check - conditional_balance.move:614-639
Missing Lifecycle Validation in Swaps - Relies on caller responsibility
futarchy_markets_operations
Missing Proposal-Escrow Validation - liquidity_interact.move implicit validation only
futarchy_oracle_actions
ClaimGrantAction Has drop Ability - oracle_actions.move:356-362 allows silent discard
State Mutation Before External Call - oracle_actions.move:561-562 marks claim before minting