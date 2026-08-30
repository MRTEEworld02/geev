# Geev Core Smart Contract

This is the core smart contract for the Geev decentralized giveaway platform, built on the Stellar blockchain using Soroban.

## Overview

The Geev Core contract enables decentralized giveaway creation and management with transparent, trustless winner selection. It implements:

- **Giveaway Creation**: Create giveaways with customizable parameters
- **Participant Registration**: Users can enter giveaways before the deadline
- **Winner Selection**: Random selection using ledger-based PRNG after giveaway ends
- **Prize Claiming**: Winners can claim their prizes once selected

## Features

### Winner Selection
Selection method is stored on the giveaway at creation (`create_giveaway` defaults
to `Random`; use `create_giveaway_with_selection` to choose explicitly):

- **Random** (`pick_winner`): ledger PRNG after `end_time`
- **FirstCome** (`finalize_first_come_winners`): first `winner_count` entrants by
  registration order; provisional winners are marked during `enter_giveaway`
- **Manual** (`finalize_manual_winners`): creator or admin supplies the winner list
- **Merit** (`finalize_merit_winners`): top reputations among participants

All methods share the same claim lifecycle (`claim_prize` / `recover_unclaimed_prize`)
once status becomes `Claimable`.

### Reputation

Reputation is a `u64` score stored under `DataKey::Reputation(Address)`.

- **Earn:** creators gain +1 on successful prize claim (`increment_reputation`).
- **Slash:** auto-suspension (flag count ≥ `FLAG_THRESHOLD`) deducts `SLASH_AMOUNT` (5)
  from the content author via `slash_reputation` (saturates at 0).
- **Restore:** `AdminContract::resolve_appeal(..., restore: true)` credits `SLASH_AMOUNT`
  back to the author.
- **Decay (on read):** `get_reputation` applies decay before returning. For each full
  `DECAY_PERIOD_SECONDS` (30 days) since `DataKey::ReputationUpdatedAt`, subtract
  `DECAY_PER_PERIOD` (1). The decayed value is persisted. Scores never go below zero.
  `min_reputation` gating uses this adjusted score.

### Protocol Fees

Fees are expressed in basis points (bps). `MAX_FEE_BPS` is `10_000` (100%).

**Resolution order at claim time** (highest precedence first):

1. **Giveaway override** — optional `fee_bps` passed at `create_giveaway` / `create_giveaway_with_selection` and stored on the `Giveaway`
2. **Per-token fee** — `DataKey::TokenFee(Address)` set via `AdminContract::set_token_fee`
3. **Global fee** — `DataKey::Fee` set at `init` or updated via `AdminContract::set_fee`
4. **Default** — `100` bps (1%) if nothing else is set

Changing global or token fees after init does **not** rewrite amounts already accrued in `DataKey::CollectedFees`. Giveaways without an override pick up the new rate on subsequent claims; giveaways with an explicit override keep that rate.

## Contract Structure

### Core Types

```rust
// Giveaway status states
pub enum GiveawayStatus {
    Active,      // Accepting entries
    Claimable,   // Winners selected, prizes claimable
    Completed,   // All prizes claimed or recovered
    Suspended,   // Governance suspension
}

// Winner selection methods
pub enum SelectionMethod {
    Random,     // Random selection via pick_winner (default)
    Manual,     // Creator/admin picks winners
    Merit,      // Highest reputation participants
    FirstCome,  // First winner_count entrants by registration order
}
```

### Storage Keys

- `Giveaway(u64)` - Store/retrieve giveaways by ID
- `ParticipantIndex(u64, u32)` - Map participant index to address
- `GiveawayCounter` - Generate unique IDs
- `HasEntered(u64, Address)` - Prevent double entry
- `Claimed(u64, Address)` - Per-winner claim record

## Functions

### `create_giveaway` / `create_giveaway_with_selection`
Create a new giveaway. `create_giveaway` stores `SelectionMethod::Random`.
`create_giveaway_with_selection` accepts an explicit `selection_method`.

### `enter_giveaway`
Add a participant to an active giveaway. For `FirstCome`, also appends the
participant to `winners` while slots remain (provisional until finalize).

### `pick_winner`
Select winners randomly when the giveaway period ends. Only valid when
`selection_method == Random`.

**Requirements:**
- Giveaway must be `Active`
- Current time must be after `end_time`
- `participant_count >= winner_count`

**Returns:** `Address` - First winner's wallet address

### `finalize_first_come_winners`
Lock in the first `winner_count` entrants (by `ParticipantIndex` order) after
`end_time`. Only valid when `selection_method == FirstCome`.

### `finalize_manual_winners` / `finalize_merit_winners`
Method-specific finalize paths for `Manual` and `Merit` giveaways (creator or admin).

### `claim_prize`
Allow a selected winner to claim their prize share while the giveaway is
`Claimable` and before `claim_deadline`.

**Requirements:**
- Giveaway status must be `Claimable`
- Claimer must be a selected winner who has not already claimed
- Claim window must not have expired

**Returns:** (void — transfers net prize after fee)

### View Functions

#### Giveaway Module

##### `get_giveaway`
Retrieve the full giveaway state by ID.

**Parameters:**
- `giveaway_id: u64` - ID of the giveaway

**Returns:** `Option<Giveaway>` - Complete giveaway data or `None` if not found

**Example:**
```rust
let giveaway = GiveawayContract::get_giveaway(env, giveaway_id);
if let Some(g) = giveaway {
    // Access g.status, g.winner_count, g.amount, etc.
}
```

##### `get_winners`
Read the list of selected winners for a giveaway.

**Parameters:**
- `giveaway_id: u64` - ID of the giveaway

**Returns:** `Vec<Address>` - List of winner addresses (empty if giveaway not found or no winners yet)

**Example:**
```rust
let winners = GiveawayContract::get_winners(env, giveaway_id);
for winner in winners.iter() {
    // Process each winner
}
```

##### `get_participants`
Read all participants who entered a giveaway, in registration order.

**Parameters:**
- `giveaway_id: u64` - ID of the giveaway

**Returns:** `Vec<Address>` - List of participant addresses (empty if giveaway not found or no participants)

**Example:**
```rust
let participants = GiveawayContract::get_participants(env, giveaway_id);
// participants[0] is the first entrant, participants[1] is the second, etc.
```

##### `has_claimed`
Check whether a specific winner has claimed their prize.

**Parameters:**
- `giveaway_id: u64` - ID of the giveaway
- `winner: Address` - Winner address to check

**Returns:** `bool` - `true` if claimed, `false` if not claimed, not a winner, or giveaway not found

**Example:**
```rust
if GiveawayContract::has_claimed(env, giveaway_id, winner.clone()) {
    // Winner has already claimed
} else {
    // Winner can still claim (if they are a winner)
}
```

#### Mutual Aid Module

##### `get_request`
Retrieve a help request by ID.

**Parameters:**
- `request_id: u64` - ID of the help request

**Returns:** `Option<HelpRequest>` - Help request data or `None` if not found

##### `get_donation`
Read the total donation amount from a specific donor for a help request.

**Parameters:**
- `request_id: u64` - ID of the help request
- `donor: Address` - Donor address

**Returns:** `i128` - Total donation amount (0 if no donation or already refunded)

**Example:**
```rust
let amount = MutualAidContract::get_donation(env, request_id, donor.clone());
```

##### `has_claimed_funds`
Check whether a help request creator has claimed the raised funds.

**Parameters:**
- `request_id: u64` - ID of the help request

**Returns:** `bool` - `true` if funds claimed, `false` otherwise

#### Admin Module

##### `get_admin`
Read the current admin address.

**Returns:** `Option<Address>` - Admin address or `None` if not initialized

##### `get_fee`
Read the global protocol fee in basis points.

**Returns:** `Option<u32>` - Fee in bps or `None` if not set

##### `get_token_fee`
Read the per-token fee override in basis points.

**Parameters:**
- `token: Address` - Token address

**Returns:** `Option<u32>` - Token-specific fee in bps or `None` if not set

##### `is_token_allowed`
Check whether a token is whitelisted for giveaway creation.

**Parameters:**
- `token: Address` - Token address to check

**Returns:** `bool` - `true` if whitelisted, `false` otherwise

**Example:**
```rust
if AdminContract::is_token_allowed(env, token.clone()) {
    // Token can be used for giveaways
}
```

##### `get_collected_fees`
Read accumulated fees collected for a specific token.

**Parameters:**
- `token: Address` - Token address

**Returns:** `i128` - Total fees collected (0 if none)

#### Profile Module

##### `get_profile`
Retrieve profile data for a wallet address.

**Parameters:**
- `user: Address` - User address

**Returns:** `Option<ProfileData>` - Profile data or `None` if not registered

##### `resolve_username`
Resolve a username to its owner's address.

**Parameters:**
- `username: String` - Username to resolve

**Returns:** `Option<Address>` - Owner address or `None` if username not registered

##### `get_reputation`
Read the reputation score for an address (applies decay-on-read).

**Parameters:**
- `user: Address` - User address

**Returns:** `u64` - Reputation score (0 if not set)

#### Governance Module

##### `get_flag_count`
Read the total number of flags for a content item.

**Parameters:**
- `content_type: ContentType` - Type of content (Giveaway or HelpRequest)
- `target_id: u64` - Content ID

**Returns:** `u32` - Total flag count

##### `has_flagged`
Check whether a user has already flagged a specific content item.

**Parameters:**
- `user: Address` - User address
- `content_type: ContentType` - Type of content
- `target_id: u64` - Content ID

**Returns:** `bool` - `true` if user has flagged, `false` otherwise

### `get_giveaway`
Retrieve giveaway details by ID.

**Parameters:**
- `giveaway_id: u64` - ID of the giveaway

**Returns:** `Option<Giveaway>` - Giveaway data or None

## Usage Examples

### Creating a Giveaway
```rust
let giveaway_id = GiveawayContract::create_giveaway(
    env.clone(),
    creator_address,
    "Free NFT Giveaway".to_string(),
    "Win one of 5 exclusive NFTs!".to_string(),
    "nft".to_string(),
    SelectionMethod::Random,
    5,           // 5 winners
    86400        // 24 hours duration
);
```

### Adding Participants
```rust
let entry_id = GiveawayContract::add_participant(
    env.clone(),
    giveaway_id,
    participant_address,
    "I'd love to win this NFT!".to_string()
);
```

### Selecting Winner (After End Time)
```rust
// Advance time beyond end_time
env.ledger().with_mut(|li| {
    li.timestamp = giveaway.end_time + 1000;
});

// Select winner
let winner_address = GiveawayContract::pick_winner(env, giveaway_id);
```

### Claiming Prize
```rust
let success = GiveawayContract::claim_prize(
    env, 
    giveaway_id, 
    winner_address
);
```

## Security Considerations

### MVP Implementation Limitations
⚠️ **Randomness Source**: Uses `env.prng()` which is ledger-based but not cryptographically secure. For production, consider using a more robust randomness source.

### Key Safeguards
- **Time-based Execution**: Winner selection only possible after `end_time`
- **Status Validation**: Prevents manipulation of completed giveaways
- **Participant Count**: Ensures at least one participant exists
- **Address Authentication**: Participants must authenticate their actions

## Testing

Run contract tests with:
```bash
cargo test
```

Tests cover:
- ✅ Giveaway creation
- ✅ Participant registration
- ✅ Winner selection with proper timing
- ✅ Error handling for edge cases
- ✅ Prize claiming functionality

## Error Handling

The contract defines specific error types:
```rust
pub enum Error {
    GiveawayNotFound = 1,
    GiveawayStillActive = 2,
    InvalidStatus = 3,
    NoParticipants = 4,
    NotCreator = 5,
    AlreadyCompleted = 6,
    InvalidIndex = 7,
    ParticipantAlreadyWinner = 8,
}
```

## Deployment

1. Build the contract:
```bash
soroban build
```

2. Deploy to testnet:
```bash
soroban deploy --network testnet
```

3. Deploy to local sandbox:
```bash
soroban deploy --network local
```

## Integration with Frontend

The contract is designed to integrate with the Geev frontend application:

- **API Alignment**: Contract methods mirror frontend concepts
- **Status Synchronization**: Giveaway states match frontend expectations
- **Error Compatibility**: Error codes map to user-friendly messages

## Future Enhancements

Planned improvements:
- 🔐 Cryptographically secure randomness
- 🔄 Multiple winner selection
- 📊 Merit-based selection algorithms
- ⚖️ Dispute resolution mechanisms
- 🛡️ Advanced participant verification