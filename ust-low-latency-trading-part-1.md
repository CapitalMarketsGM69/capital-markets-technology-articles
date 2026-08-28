# Designing a Low-Latency Electronic Trading Architecture for U.S. Treasuries

## Part 1: Markets, Rates & Architecture


```text
Part 1: Markets, Rates & Architecture
Part 2: Java, RxJava & Million-Event Processing
```
A technical primer on event-driven rates trading, market data, risk, and Java-based processing




# Part 1: Markets, Rates & Architecture
Electronic trading in U.S. Treasuries is fundamentally an event-driven problem. A trading system continuously receives changes in prices, yields, quantities, trades, curve relationships, futures markets, and other correlated rates instruments. The objective is to process those events quickly and predictably enough to update fair value, identify a signal, apply risk controls, and place or modify orders before the market moves.

Low latency is therefore not simply about writing fast code. It is about designing the entire event lifecycle so that market-data ingestion, book building, analytics, strategy, risk, and execution operate with minimal and predictable delay.

## 1. A Simplified U.S. Treasury Trading Flow
```text
Venue Market Data
       |
       v
+-------------------+
| Binary Feed       |
| Decoder           |
+-------------------+
       |
       v
+-------------------+
| Normalized Market |
| Data Model        |
+-------------------+
       |
       +-----------> Order Book
       |
       +-----------> Treasury Curve
       |
       +-----------> Analytics
                         |
                         v
                +----------------+
                | Fair Value /   |
                | Signal Engine  |
                +----------------+
                         |
                         v
                +----------------+
                | Pre-Trade Risk |
                +----------------+
                         |
                         v
                +----------------+
                | Execution Algo |
                +----------------+
                         |
                         v
                   Trading Venue
```
## 2. U.S. Treasury Instruments
A rates platform may trade several Treasury maturities and compare them with related instruments.

2-Year, 3-Year, 5-Year, 7-Year, 10-Year, 20-Year and 30-Year U.S. Treasuries

Treasury futures

SOFR futures

Interest-rate swaps and OIS curves

Other Treasury maturities and related spread products

## 3. Treasury Price Representation
Treasury prices are often quoted in 32nds. For example, 99-16 means 99 + 16/32 = 99.500000, while 99-17 means 99 + 17/32 = 99.53125.

In a latency-sensitive Java engine, it is often preferable to represent prices as integer ticks instead of floating-point numbers.

```text
// 99-17 represented in 32nd ticks
long priceTicks = (99 * 32L) + 17;   // 3185
```
## 4. Core Market-Data Event
A market-data event should be compact. In production, object reuse or off-heap structures may be used to reduce allocation pressure, but the following immutable object is useful for illustration.

```text
public final class TreasuryTick {

    public final long timestampNanos;
    public final int instrumentId;
    public final long bidPriceTicks;
    public final long askPriceTicks;
    public final long bidQty;
    public final long askQty;

    public TreasuryTick(
            long timestampNanos,
            int instrumentId,
            long bidPriceTicks,
            long askPriceTicks,
            long bidQty,
            long askQty) {

        this.timestampNanos = timestampNanos;
        this.instrumentId = instrumentId;
        this.bidPriceTicks = bidPriceTicks;
        this.askPriceTicks = askPriceTicks;
        this.bidQty = bidQty;
        this.askQty = askQty;
    }
}
```
## 5. Midpoint, Microprice and Order-Book Imbalance
The midpoint is the average of the best bid and offer. A microprice goes one step further by weighting the two prices by opposite-side quantity, which can provide a simple indication of short-term pressure.

```text
public static double midpoint(TreasuryTick t) {
    return (t.bidPriceTicks + t.askPriceTicks) / 2.0;
}

public static double microPrice(TreasuryTick t) {
    long totalQty = t.bidQty + t.askQty;

    if (totalQty == 0) {
        return midpoint(t);
    }

    return (
        t.askPriceTicks * (double) t.bidQty +
        t.bidPriceTicks * (double) t.askQty
    ) / totalQty;
}

public static double imbalance(TreasuryTick t) {
    long totalQty = t.bidQty + t.askQty;

    if (totalQty == 0) {
        return 0.0;
    }

    return (t.bidQty - t.askQty) / (double) totalQty;
}
```
An imbalance near +1 indicates strong bid-side quantity; an imbalance near -1 indicates strong offer-side quantity. Neither microprice nor imbalance should be treated as a complete trading strategy, but they are useful features in a broader signal model.

## 6. Rates Trading Is About Yield and Risk
A Treasury trading engine must think in terms of interest-rate risk, not just price. Treasury prices and yields generally move in opposite directions. As a result, a serious rates platform typically maintains price, yield, duration, DV01, spread, and curve relationships.

```text
Example curve snapshot:

2Y yield   = 3.61%
5Y yield   = 3.72%
10Y yield  = 3.94%
30Y yield  = 4.55%

2s10s = 10Y yield - 2Y yield
      = 3.94% - 3.61%
      = 0.33% = 33 basis points
```
## 7. DV01 and Risk-Normalized Hedging
DV01 approximates the change in market value for a one-basis-point move in yield. It is a core measure for comparing and hedging positions across maturities.

DV01 ≈ Modified Duration × Market Value × 0.0001

A strategy that is long $25,000 of 10-year DV01 might hedge approximately $25,000 of DV01 in another maturity, rather than simply using an equal face amount.

## 8. Relative-Value and Curve Signals
Treasury algorithms frequently operate on relationships between maturities. A simple educational example is a 5s10s30s butterfly.

```text
double butterfly =
        (2 * y10) - y5 - y30;
```
A production strategy would also account for DV01 weighting, transaction costs, bid/ask spread, liquidity, carry, roll-down, funding, execution risk, and hedging risk.

## 9. Pre-Trade Risk Is Part of the Hot Path
A signal should never go directly to an order gateway. The execution path should contain fast, deterministic pre-trade controls.

```text
Signal
  |
  v
Position Check
  |
  v
DV01 Check
  |
  v
Notional / Order Size
  |
  v
Price Collar
  |
  v
Order Rate Limit
  |
  v
Kill Switch
  |
  v
Order Gateway
```
## 10. Order State Management
Orders should move through an explicit state machine so that asynchronous execution reports cannot mutate state unpredictably.

```text
NEW
 |
 v
PENDING_NEW
 |
 +------> REJECTED
 |
 v
LIVE
 |
 +------> PARTIALLY_FILLED
 |
 +------> FILLED
 |
 +------> PENDING_CANCEL
             |
             v
          CANCELLED
```
## 11. Market-Making and Inventory Skew
A market-making engine may calculate fair value, choose a target spread, and then adjust its bid and offer based on inventory, volatility, liquidity, and current risk. If the system is already long duration, it may make its bid less aggressive and its offer more aggressive to encourage inventory reduction.

## 12. What 'Low Latency' Really Means
Throughput and latency are different. A system might process two million messages per second and still experience occasional multi-millisecond stalls. For trading, tail latency often matters as much as average throughput.

p50 latency

p90 latency

p99 latency

p99.9 latency

maximum latency

[Continue to Part 2](ust-low-latency-trading-part-2.md)
