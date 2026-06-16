# PositionSizerStyleCbot

A cTrader Automate cBot that combines a draggable chart-based trade planner with a compact execution panel for risk-based position sizing, pending order placement, and live visual trade planning.

This project is designed for discretionary traders who want the speed of chart trading with tighter control over risk, stop-loss distance, take-profit distance, and execution mode. It supports instant execution, pending orders, and stop-limit orders, while continuously recalculating trade size from the selected risk amount.

## What it does

PositionSizerStyleCbot places and manages a three-line trade plan on the chart:

- **Entry line**
- **Stop-loss line**
- **Take-profit line**

From those three prices, the cBot:

- infers trade direction automatically,
- calculates stop-loss and take-profit distances in pips,
- calculates the intended risk amount from either `% risk` or fixed `money risk`,
- computes trade volume in units,
- shows a live RR preview,
- validates the plan against configurable safety fuses,
- places market, limit, stop, or stop-limit orders depending on the selected mode.

## Core features

### 1. Chart-based planning

The cBot draws three horizontal lines on the active chart:

- `PS_ENTRY`
- `PS_SL`
- `PS_TP`

You can drag the SL and TP lines at any time. In non-instant modes, you can also drag the entry line.

### 2. Panel-based editing

The on-chart panel allows direct numeric input for:

- Entry price
- Stop-loss price
- Take-profit price
- Risk %
- Fixed money risk
- Commission per lot

The panel updates the chart lines immediately, so numeric edits and line dragging stay synchronized.

### 3. Input edit-lock protection

A temporary edit lock is applied while typing into panel text boxes. This prevents `OnTick()` / `RefreshUi()` updates from instantly overwriting partially typed values.

Current setting:

- `InputEditLock = 2000 ms`

This makes the panel much easier to use during fast markets or high tick frequency.

### 4. Multiple execution modes

The cBot supports three entry modes:

- **Instant**: executes immediately at Ask for buys or Bid for sells.
- **Pending**: automatically chooses stop order or limit order based on entry price relative to the current market.
- **StopLimit**: places a stop-limit order using the configured stop-limit range.

### 5. Flexible trade button placement

Trade controls can appear in one of three layouts:

- `PanelOnly`
- `LineOnly`
- `PanelAndLine`

This allows either a compact panel workflow or a line-centric chart workflow.

### 6. Optional spread and commission inclusion

Risk sizing can include:

- current spread,
- round-turn commission per lot.

This allows a more conservative size calculation when you want the full execution cost included in the risk budget.

### 7. Safety fuses

The cBot includes validation rules that can block execution when trade conditions do not match your plan:

- Max spread filter
- Minimum entry-to-SL distance
- Maximum entry-to-SL distance
- Maximum allowed risk %
- Pending entry too close to market
- Zero-volume rejection
- Invalid SL / TP rejection

### 8. Live chart labels

Optional text labels are drawn near the trade lines showing:

- direction,
- entry price,
- calculated volume,
- SL distance,
- TP distance,
- estimated risk,
- estimated reward.

## How direction is inferred

Direction is inferred entirely from the relationship between entry and stop-loss:

- If `SL < Entry`, the trade is treated as **Buy**.
- If `SL > Entry`, the trade is treated as **Sell**.

That means you do not manually choose buy or sell from a dropdown. The line placement defines the trade direction.

## How risk is chosen

The cBot supports two risk modes:

### Risk % mode

If `Money Risk = 0`, the cBot calculates risk from account balance:

```text
RiskAmount = Account.Balance * (RiskPercent / 100)
```

### Fixed money mode

If `Money Risk > 0`, the cBot uses that value directly:

```text
RiskAmount = MoneyRisk
```

The **USE %** button resets fixed money risk back to zero, which reactivates percentage-based risk sizing.

## How volume is calculated

The cBot calculates stop distance first:

```text
SL pips = abs(EntryPrice - StopLossPrice) / Symbol.PipSize
```

Then it builds an effective risk-per-unit cost:

```text
effectivePips = slPips + spreadPips (optional)
commissionPerUnit = CommissionPerLotRoundTurn / Symbol.LotSize (optional)
costPerUnit = effectivePips * Symbol.PipValue + commissionPerUnit
rawVolume = RiskAmount / costPerUnit
finalVolume = Symbol.NormalizeVolumeInUnits(rawVolume, RoundingMode.Down)
```

### Important note on gold / XAU sizing

This build uses `Symbol.PipValue` **directly** in the sizing formula.

That change was made to address the gold sizing issue where volume could become roughly **100x too large** when pip value was previously divided by `Symbol.LotSize` a second time. In practical terms:

- the previous logic could oversize symbols such as XAUUSD,
- the current logic keeps the pip-value term in the same scale as the normalized volume calculation,
- commission is still converted from per-lot to per-unit using `CommissionPerLotRoundTurn / Symbol.LotSize`.

If you trade metals, indices, or CFDs with non-forex contract conventions, always verify the panel output on a demo account before relying on live execution.

## Parameters

## Risk group

| Parameter | Purpose |
|---|---|
| `Risk %` | Percentage of account balance to risk when fixed money risk is zero. |
| `Money Risk (0 = use %)` | Fixed monetary risk amount. Overrides percentage mode when greater than zero. |
| `Include Spread In Risk` | Adds current spread to the effective SL distance used for sizing. |
| `Include Commission In Risk` | Adds configured commission cost to the sizing formula. |
| `Commission Per Lot Round Turn` | Round-turn commission cost per lot. Used only when commission inclusion is enabled. |

## Trading group

| Parameter | Purpose |
|---|---|
| `Entry Mode` | Initial execution mode: `Instant`, `Pending`, or `StopLimit`. |
| `Trade Button Placement` | Controls whether trade buttons appear on the panel, near the line, or both. |
| `Do Not Apply SL` | Sends the trade without a stop-loss value. Use with caution. |
| `Do Not Apply TP` | Sends the trade without a take-profit value. |
| `Expiry Minutes` | Expiry time for pending or stop-limit orders. `0` disables expiry. |
| `Label` | Order label used for placed trades/orders. |
| `Comment` | Optional order comment. |

## Fuses group

| Parameter | Purpose |
|---|---|
| `Max Spread (pips, 0=off)` | Blocks execution if current spread exceeds the limit. |
| `Min Entry-SL Distance (pips, 0=off)` | Blocks execution if SL distance is too small. |
| `Max Entry-SL Distance (pips, 0=off)` | Blocks execution if SL distance is too large. |
| `Max Risk % (0=off)` | Prevents percentage risk input from exceeding the configured ceiling. |

## Defaults group

| Parameter | Purpose |
|---|---|
| `Initial SL Pips` | Default SL distance at startup. |
| `Initial TP Pips` | Default TP distance at startup. |

## Visual group

| Parameter | Purpose |
|---|---|
| `Line Thickness` | Thickness of entry, SL, and TP lines. |
| `Show Panel` | Enables the on-chart panel. |
| `Show Chart Labels` | Enables text labels near the lines. |
| `Start Minimized` | Starts the panel in collapsed mode. |

## Panel controls

The panel contains these main controls:

- **TRADE**: executes the current trade plan.
- **FLIP**: mirrors SL/TP to the other side of entry and reverses inferred direction.
- **USE %**: clears fixed money risk and returns to percentage risk mode.
- **MODE**: cycles through `Instant -> Pending -> StopLimit -> Instant`.
- **SPR ON / OFF**: toggles spread inclusion in risk sizing.
- **COM ON / OFF**: toggles commission inclusion in risk sizing.
- **+ / -**: collapses or expands the panel.

## Execution logic

### Instant mode

In instant mode:

- Entry is automatically pinned to the live market.
- Buy entry uses `Symbol.Ask`.
- Sell entry uses `Symbol.Bid`.
- Entry textbox is read-only.

This mode is intended for immediate execution while still allowing SL and TP to be dragged interactively.

### Pending mode

In pending mode, the cBot determines whether the pending order should be a stop or a limit order.

For buys:

- `Entry > current Ask` -> buy stop
- `Entry < current Ask` -> buy limit

For sells:

- `Entry < current Bid` -> sell stop
- `Entry > current Bid` -> sell limit

### StopLimit mode

In stop-limit mode, the cBot places a stop-limit order with the configured stop-limit range:

```text
StopLimitRangePips = 2.0
```

This value is currently internal, not parameterized in the UI.

## Validation rules

Before any order is placed, the cBot validates the trade plan.

Execution is blocked if any of the following is true:

- SL distance is zero or invalid.
- TP distance is invalid while TP is required.
- Current spread exceeds the configured maximum.
- Entry-to-SL distance is below the minimum fuse.
- Entry-to-SL distance is above the maximum fuse.
- Risk % exceeds the configured maximum when using percentage mode.
- Calculated volume is zero.
- Pending entry is too close to the current market.

Validation status is shown on the panel and also printed when execution fails.

## Typical workflow

### Market execution workflow

1. Start the cBot on the desired chart.
2. Drag the SL and TP lines into place.
3. Set risk using either `%` or fixed `$`.
4. Optionally enable spread and/or commission inclusion.
5. Confirm the live volume and RR shown on the panel.
6. Press **TRADE**.

### Pending order workflow

1. Press **MODE** until the cBot is in `Pending`.
2. Move the entry line to the desired pending price.
3. Adjust SL and TP.
4. Confirm risk and volume.
5. Press **TRADE**.

### Direction flip workflow

1. Build an initial trade plan.
2. Press **FLIP**.
3. The cBot mirrors SL and TP across entry.
4. Direction is recalculated automatically.

## UI synchronization behavior

The cBot keeps chart lines and panel inputs synchronized in both directions.

### Line -> panel

When you drag a line, the displayed values in the panel update.

### Panel -> line

When you type a valid numeric value in the panel, the related line moves immediately.

### Typing protection

While a textbox is being edited, the 2000 ms edit lock prevents the next refresh cycle from instantly forcing the field back to the previously formatted value. This is especially important because `OnTick()` continues to refresh UI state during active market movement.

## Known implementation details

- The bot uses **volume in units**, not lots, internally.
- Displayed volume on the panel is currently the normalized unit volume.
- For symbols with non-standard contract specifications, the relationship between units and broker-visible lot size may differ from major forex pairs.
- Gold, indices, and some CFDs should always be tested on demo first.
- In `Instant` mode, entry is controlled by market price and cannot be typed manually.
- The stop-limit range is currently fixed at `2.0` pips in code.

## Suggested future improvements

Potential upgrades that fit the existing design:

- Show both **units** and **lots/quantity** in the panel.
- Expose stop-limit range as a parameter.
- Add presets for common risk values.
- Add hotkeys for `TRADE`, `FLIP`, and mode switching.
- Add optional breakeven / partial-close helpers.
- Add broker-symbol diagnostics for metals and CFDs.
- Add a dedicated status color system for ready / warning / blocked states.

## Troubleshooting

### I cannot type into the panel cleanly

Check that the build includes:

```csharp
private static readonly TimeSpan InputEditLock = TimeSpan.FromMilliseconds(2000);
```

If needed, increase it further for very fast feeds.

### Gold size still looks wrong

Verify the following carefully:

- The broker's XAU symbol contract specification.
- `Symbol.PipSize`
- `Symbol.PipValue`
- `Symbol.LotSize`
- Whether the broker displays quantity, lots, ounces, or units in the trade ticket.
- Whether your commission setting is truly round-turn and per lot.

The cBot now uses `Symbol.PipValue` directly in the risk formula, which is the main fix for the reported gold oversizing issue.

### Pending order is rejected as too close

Move the entry line farther from the current market price. The cBot intentionally blocks pending entries that are effectively on top of live price.

### Volume becomes zero

Possible causes:

- risk amount too small,
- SL too wide,
- symbol pip value too large relative to the chosen risk,
- spread and commission filters making effective cost too high,
- broker minimum volume increment after normalization.

## Installation

1. Open **cTrader Automate**.
2. Create a new cBot or open the existing source file.
3. Replace the contents with `PositionSizerStyleCbot.cs`.
4. Build the project.
5. Attach the cBot to the intended symbol chart.
6. Configure parameters.
7. Start the bot.

## Recommended testing checklist

Before live use:

- Test on a demo account.
- Check a forex pair and one metal symbol.
- Compare expected risk vs actual ticket size.
- Test `Instant`, `Pending`, and `StopLimit` modes.
- Test spread inclusion on and off.
- Test commission inclusion on and off.
- Verify expiry handling for pending orders.
- Verify fuse behavior by intentionally violating each fuse once.

## Changelog summary

### Recent changes

- Added textbox edit-lock protection to stop UI refresh from overwriting panel input while typing.
- Increased input edit-lock duration from `800 ms` to `2000 ms`.
- Updated sizing logic to use `Symbol.PipValue` directly in the pip-cost calculation.
- Corrected the previously reported oversizing behavior on gold/XAU-style symbols.

## Disclaimer

This cBot is a trading utility, not a guarantee of execution quality or risk control under all broker conditions. Always validate the calculated position size, execution mode, and symbol contract specification on a demo account before using it in a live environment.
