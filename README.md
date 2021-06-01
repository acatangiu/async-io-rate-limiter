# ratelimit-asyncio

Provides a rate limiter written in Rust useful for IO operations running in
an async event-loop on a single thread, which need to be throttled.

## Design

This crate is build around two abstractions:

- TokenBucket
- IoRateLimiter

### TokenBucket

A `TokenBucket` object provides a low level interface to a passive token
bucket implementation with configurable capacity, refill-rate and one-time
budget.

Its interface is very simple with a public constructor for configuration,
a `pub fn consume(&mut self, tokens: u64)` principal function, and various
getters.

The `TokenBucket` is described as _passive_ because it is exclusively
driven by calls to its `consume()` API. It manages its internal state - most
importantly, both consuming and auto-replenishing the _current bucket
token budget_ - transparently as a side-effect of _consume_ operations.

None of its API calls are blocking; a _consume_ operation can either fail
or succeed, the caller having the freedom to implement the upper layers'
logic based on said result.

#### API
```rust
pub fn new(size: u64, one_time_burst: u64, refill_time_ms: u64) -> Self
```
Creates a TokenBucket of `size` total capacity that takes `refill_time_ms`
milliseconds to go from zero tokens to total capacity. The `one_time_burst`
is initial extra credit on top of total capacity, that does not replenish
and which can be used for an initial burst of data.

```rust
pub fn consume(&mut self, tokens: u64) -> ConsumptionResult
```
Attempts to consume number of tokens (specified by `tokens`) from the
bucket and returns whether the action succeeded.
Internally, this function also auto-replenishes the _token budget_
based on timestamp deltas. As such, the TokenBucket can be used by
exclusively calling its `consume()` API.

```rust
pub fn force_replenish(&mut self, tokens: u64)
```
"Manually" adds tokens to bucket.

```rust
pub fn capacity(&self) -> u64
```
Returns the capacity of the token bucket.

```rust
pub fn one_time_burst(&self) -> u64
```
Returns the remaining one time burst budget.

```rust
pub fn refill_time_ms(&self) -> u64
```
Returns the time in milliseconds required to to completely fill the bucket.

```rust
pub fn budget(&self) -> u64
```
Returns the current budget (one time burst allowance notwithstanding).


### IoRateLimiter

`IoRateLimiter` is a specialized rate-limiter implementation that works on
both **IO bandwidth** and **IO #ops / s** limiting.

Bandwidth (bytes/s) and ops/s limiting can be used at the same time or
individually. Internally it uses two `TokenBucket`s, one for _bandwidth_ and
the other  for _ops_ limiting.

It is designed to be integrated in a low-level `Linux epoll/poll/select`
event loop, and as such it uses a single `TimerFd` to refresh either or both
token buckets.

Its internal buckets are _passively_ replenished as they're being used
(as part of `consume()` operations). A timer is enabled and used to _actively_
replenish the token buckets when limiting is in effect and `consume()`
operations are disabled/not happening.

`RateLimiter`s will generate events on the FDs provided by their `AsRawFd`
trait implementation. These events are meant to be consumed by the user of
this struct. On each such event, the user must call the `event_handler()`
method.

#### Behavior

The rate limiter starts off as 'unblocked' with two token buckets configured with the values passed in the RateLimiter::new() constructor. All subsequent accounting is done independently for each token bucket based on the TokenType used. If any of the buckets runs out of budget, the limiter goes in the 'blocked' state. At this point an internal timer is set up which will later 'wake up' the user in order to retry sending data. The 'wake up' notification will be dispatched as an event on the FD provided by the AsRawFD trait implementation.

The contract is that the user shall also call the event_handler() method on receipt of such an event.

The token buckets are replenished when a called consume() doesn't find enough tokens in the bucket. The amount of tokens replenished is automatically calculated to respect the complete_refill_time configuration parameter provided by the user. The token buckets will never replenish above their respective size.

Each token bucket can start off with a one_time_burst initial extra capacity on top of their size. This initial extra credit does not replenish and can be used for an initial burst of data.

The granularity for 'wake up' events when the rate limiter is blocked is currently hardcoded to 100 milliseconds.

#### Limitations

This rate limiter implementation relies on the Linux's `timerfd` so its usage
is limited to Linux systems.

Another particularity of this implementation is that it is not self-driving.
It is meant to be used in an external low-level `epoll/poll/select` event
loop and thus implements the `AsRawFd` trait and provides an event-handler
as part of its API. This event-handler needs to be called by the user on
every event on the rate limiter's `AsRawFd` FD.

#### API

```rust
pub fn new(
    bytes_total_capacity: u64,
    bytes_one_time_burst: u64,
    bytes_complete_refill_time_ms: u64,
    ops_total_capacity: u64,
    ops_one_time_burst: u64,
    ops_complete_refill_time_ms: u64
) -> Result<Self>
```
Creates a new Rate Limiter that can limit on both bytes/s and ops/s.

If either bytes or ops `size` or `refill_time` are **zero**, the limiter is
**disabled** for that respective token type.

```rust
pub fn consume(&mut self, tokens: u64, token_type: TokenType) -> bool
```
Attempts to consume `tokens` number of tokens of type `token_type` (bytes/ops)
and returns whether it succeeded.

If rate limiting is disabled on provided `token_type`, this function will
always succeed.

```rust
pub fn manual_replenish(&mut self, tokens: u64, token_type: TokenType)
```
Adds `tokens` of `token_type` to their respective bucket.

Can be used to manually add tokens to a bucket. Useful for reverting a
`consume()` if needed.

```rust
pub fn is_blocked(&self) -> bool
```
Returns whether this rate limiter is blocked.

The limiter 'blocks' when a `consume()` operation fails because there was not
enough budget for it. An event will be generated on the exported FD when the
limiter 'unblocks'.

```rust
pub fn event_handler(&mut self) -> Result<(), Error>
```
This function needs to be called every time there is an event on the FD
provided by this object's `AsRawFd` trait implementation.

## Usage

TODO: This section describes how the crate is used.

Some questions that might help in writing this section:
- What traits do users need to implement?
- Does the crate have any default/optional features? What is each feature
  doing?
- Is this crate used by other rust-vmm components? If yes, how?

## Examples

```rust
use RateLimiter::*;

let bytes = 1000;
let bytes_burst = 0;
let bytes_refill_time = 1000;
let ops = 0;
let ops_burst = 0;
let ops_refill_time = 0;
// rate limiter with limit of 1000 bytes/s an unlimited ops.
let mut l = RateLimiter::new(
    bytes, bytes_burst, bytes_refill_time,
    ops, ops_burst, ops_refill_time
).unwrap();

// limiter should not be blocked
assert!(!l.is_blocked());

// ops/s limiter should be disabled so consume(whatever) should work
assert!(l.consume(u64::max_value(), TokenType::Ops));

// do full 1000 bytes
assert!(l.consume(1000, TokenType::Bytes));
// try and fail on another 100
assert!(!l.consume(100, TokenType::Bytes));
// since consume failed, limiter should be blocked now
assert!(l.is_blocked());
// wait half the timer period
thread::sleep(Duration::from_millis(REFILL_TIMER_INTERVAL_MS / 2));
// limiter should still be blocked
assert!(l.is_blocked());
// wait the other half of the timer period
thread::sleep(Duration::from_millis(REFILL_TIMER_INTERVAL_MS / 2));
// the timer_fd should have an event on it by now
assert!(l.event_handler().is_ok());
// limiter should now be unblocked
assert!(!l.is_blocked());
// try and succeed on another 100 bytes this time
assert!(l.consume(100, TokenType::Bytes));
```

## License

**!!!NOTICE**: The BSD-3-Clause license is not included in this template.
The license needs to be manually added because the text of the license file
also includes the copyright. The copyright can be different for different
crates. If the crate contains code from CrosVM, the crate must add the
CrosVM copyright which can be found
[here](https://chromium.googlesource.com/chromiumos/platform/crosvm/+/master/LICENSE).
For crates developed from scratch, the copyright is different and depends on
the contributors.
