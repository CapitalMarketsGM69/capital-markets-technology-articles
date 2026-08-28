# Designing a Low-Latency Electronic Trading Architecture for U.S. Treasuries

## Part 2: Java, RxJava & Million-Event Processing

[Back to Part 1](ust-low-latency-trading-part-1.md)

Part 2 moves from market structure and architecture into implementation. RxJava can be very useful for high-throughput event processing, fan-out, enrichment, monitoring, and strategy pipelines. However, an ultra-low-latency order-entry loop should be designed carefully because every scheduler hop, queue, allocation, and blocking call adds variance.

## 1. RxJava Dependency
```text
<dependency>
    <groupId>io.reactivex.rxjava3</groupId>
    <artifactId>rxjava</artifactId>
    <version>3.1.10</version>
</dependency>
```
## 2. Analytics Result
```text
public record TreasuryAnalyticsResult(
        int instrumentId,
        long timestampNanos,
        double midpoint,
        double microPrice,
        double imbalance,
        long spreadTicks) {
}
```
## 3. Analytics Calculation
```text
public final class TreasuryAnalytics {

    public static TreasuryAnalyticsResult calculate(TreasuryTick t) {

        double midpoint =
                (t.bidPriceTicks + t.askPriceTicks) / 2.0;

        long totalQty =
                t.bidQty + t.askQty;

        double microPrice =
                totalQty == 0
                ? midpoint
                : (
                    t.askPriceTicks * (double)t.bidQty +
                    t.bidPriceTicks * (double)t.askQty
                  ) / totalQty;

        double imbalance =
                totalQty == 0
                ? 0
                : (t.bidQty - t.askQty)
                  / (double) totalQty;

        long spread =
                t.askPriceTicks - t.bidPriceTicks;

        return new TreasuryAnalyticsResult(
                t.instrumentId,
                t.timestampNanos,
                midpoint,
                microPrice,
                imbalance,
                spread
        );
    }
}
```
## 4. Basic RxJava Event Pipeline
```text
Flowable<TreasuryTick> marketData =
        Flowable.fromIterable(ticks);

marketData
    .filter(t -> t.bidPriceTicks > 0)
    .filter(t -> t.askPriceTicks > t.bidPriceTicks)
    .map(TreasuryAnalytics::calculate)
    .filter(a -> Math.abs(a.imbalance()) > 0.25)
    .subscribe(System.out::println);
```
## 5. Processing One Million Treasury Updates
The following benchmark generates one million simulated events and pushes them through an RxJava Flowable. It reports both elapsed time and processing rate.

```text
import io.reactivex.rxjava3.core.Flowable;

import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.AtomicLong;

public class MillionTreasuryTicks {

    private static final int RECORD_COUNT = 1_000_000;

    public static void main(String[] args) {

        AtomicLong processed = new AtomicLong();

        long start = System.nanoTime();

        Flowable
            .range(0, RECORD_COUNT)
            .map(MillionTreasuryTicks::createTick)
            .filter(t -> t.askPriceTicks > t.bidPriceTicks)
            .map(TreasuryAnalytics::calculate)
            .filter(result -> Math.abs(result.imbalance()) > 0.10)
            .doOnNext(x -> processed.incrementAndGet())
            .blockingSubscribe();

        long elapsedNanos = System.nanoTime() - start;
        double seconds = elapsedNanos / 1_000_000_000.0;

        System.out.printf("Input records       : %,d%n", RECORD_COUNT);
        System.out.printf("Signals generated   : %,d%n", processed.get());
        System.out.printf("Elapsed             : %.4f sec%n", seconds);
        System.out.printf("Processing rate     : %,.0f records/sec%n",
                RECORD_COUNT / seconds);
    }

    private static TreasuryTick createTick(int i) {

        ThreadLocalRandom random = ThreadLocalRandom.current();

        long bid = 3180 + random.nextInt(20);
        long ask = bid + 1;

        long bidQty = random.nextLong(
                1_000_000,
                200_000_000);

        long askQty = random.nextLong(
                1_000_000,
                200_000_000);

        return new TreasuryTick(
                System.nanoTime(),
                10,
                bid,
                ask,
                bidQty,
                askQty);
    }
}
```
## 6. Throughput Is Not the Same as Latency
A million-record benchmark proves throughput, not low latency. To understand whether a trading engine is actually low latency, measure per-event timing and especially the tail of the distribution.

```text
long latencyNanos =
        System.nanoTime() - tick.timestampNanos;
```
In a production benchmark, record latency in a histogram and inspect p50, p90, p99, p99.9, and maximum latency.

## 7. RxJava Backpressure
If the market-data producer runs faster than downstream consumers, queues can grow indefinitely. RxJava Flowable provides backpressure support.

```text
Flowable<TreasuryTick> stream =
    Flowable.create(
        emitter -> {
            while (!emitter.isCancelled()) {
                TreasuryTick tick = receiveMarketData();
                emitter.onNext(tick);
            }
        },
        BackpressureStrategy.LATEST
    );
```
BackpressureStrategy.LATEST can be appropriate for some analytics or monitoring streams, but dropping events can be unsafe for deterministic order-book construction unless the market-data protocol provides appropriate sequence and recovery semantics.

## 8. Separate Trading-Critical and Non-Critical Streams
```text
                     Market Data
                         |
                         v
                     Feed Handler
                         |
          +--------------+--------------+
          |                             |
          v                             v
 Trading-Critical                 Non-Critical
     Pipeline                       Pipeline
          |                             |
    Book Builder                     RxJava
          |                       Analytics
     Strategy                       Metrics
          |                         Logging
       Risk                        Monitoring
          |
     Execution
```
Logging, database writes, dashboards, telemetry, and historical persistence should not block the trading-critical path.

## 9. Scheduler and Thread-Handoff Costs
A common RxJava mistake is to insert scheduler boundaries throughout the pipeline. Every observeOn introduces a queue and a thread handoff.

```text
ExecutorService marketExecutor =
        Executors.newSingleThreadExecutor();

Scheduler marketScheduler =
        Schedulers.from(marketExecutor);

marketData
    .observeOn(marketScheduler, false, 4096)
    .map(TreasuryAnalytics::calculate)
    .subscribe(strategy::onAnalytics);
```
For the most latency-sensitive pipeline, keeping feed normalization, analytics, and strategy logic on one owning thread may produce more predictable latency than repeated scheduler hops.

## 10. A Simple Educational Trading Signal
```text
public enum Side {
    BUY,
    SELL,
    NONE
}

public static Side signal(TreasuryAnalyticsResult x) {

    double displacement =
            x.microPrice() - x.midpoint();

    if (x.imbalance() > 0.35 && displacement > 0) {
        return Side.BUY;
    }

    if (x.imbalance() < -0.35 && displacement < 0) {
        return Side.SELL;
    }

    return Side.NONE;
}
```
This is deliberately simplistic and should be treated as a programming example, not a production strategy.

## 11. Pre-Trade Risk Example
```text
public boolean riskApproved(Order order) {

    if (Math.abs(position + order.quantity()) > maxPosition) {
        return false;
    }

    if (Math.abs(currentDv01 + order.dv01()) > maxDv01) {
        return false;
    }

    if (order.quantity() > maxOrderSize) {
        return false;
    }

    return true;
}
```
## 12. Reduce Allocation in the Hot Path
Object allocation and garbage collection introduce variance. A production low-latency system may use preallocated event objects, primitive fields, ring buffers, or off-heap structures.

Prefer integer or fixed-point price representation where appropriate

Avoid BigDecimal in per-tick hot-path calculations unless the precision requirement demands it

Avoid building Strings for every market event

Reuse buffers and event containers when possible

Move reporting and formatting away from the trading thread

## 13. Avoid Lock Contention
A single-owner model is often simpler and faster than sharing mutable order books across many threads. If one processing thread owns a book and its related strategy state, many locks can be eliminated.

```text
Market Thread
     |
     +-- Order Book
     +-- Analytics
     +-- Strategy
     +-- Position Cache
```
## 14. CPU Affinity and Predictability
At the more advanced end of low-latency engineering, critical threads may be pinned to dedicated CPU cores to reduce scheduler jitter, improve cache locality, and reduce thread migration.

```text
CPU 2 -> Feed Handler
CPU 3 -> 2Y / 5Y Strategy
CPU 4 -> 10Y Strategy
CPU 5 -> 30Y Strategy
CPU 6 -> Execution
CPU 7 -> Risk / Position
```
## 15. Logging and Persistence
Synchronous logging and database writes can dominate latency at high event rates. A more robust pattern is to publish telemetry or journal records asynchronously.

```text
Trading Pipeline
      |
      +----------> Strategy / Execution
      |
      +----------> Lock-Free Telemetry Queue
                         |
                         v
                 Async Monitoring / Persistence
```
## 16. Replay and Deterministic Analysis
Replay is essential for production support, testing, model validation, and incident analysis. A platform should be able to capture market events, orders, execution reports, risk state, and strategy state and later replay them in sequence.

What did the algorithm know at the time?

Why did it quote or cancel?

What was the order book?

What was the position and DV01?

Which risk check accepted or rejected the action?

## 17. Practical Java Technology Stack
Java 21+

RxJava

LMAX Disruptor

Aeron

Agrona

SBE

Netty

Chronicle Queue

HdrHistogram

JMH

## 18. Final Architecture
```text
                     U.S. TREASURY VENUES
                              |
                              v
                     Market Data Gateway
                              |
                    Binary Decode / Normalize
                              |
                              v
                 +--------------------------+
                 | Low-Latency Event Bus    |
                 +--------------------------+
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
     Order Books                       RxJava Stream
          |                                 |
          v                         Analytics / Monitoring
     Rates Analytics                         |
          |                            Persistence
          v
     Fair Value Engine
          |
          v
      Algo Strategy
          |
          +---- Curve
          +---- Microprice
          +---- Imbalance
          +---- Inventory
          +---- Volatility
          |
          v
      Pre-Trade Risk
          |
          +---- Position
          +---- DV01
          +---- Notional
          +---- Price Collar
          +---- Kill Switch
          |
          v
      Execution Engine
          |
          v
      Order Gateway
          |
          v
                   U.S. TREASURY VENUE
```
## Conclusion
Low-latency U.S. Treasury trading is not simply a matter of making Java process a large number of records. The real engineering challenge is achieving predictable end-to-end behavior across market data, order books, rates analytics, strategy, risk, and execution. RxJava can be a valuable component of that architecture, especially for high-throughput event processing and non-blocking pipelines, but the most latency-sensitive path should be kept as lean, deterministic, and allocation-conscious as possible.


End of Part 1 and Part 2