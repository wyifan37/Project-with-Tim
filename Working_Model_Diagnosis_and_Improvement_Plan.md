# Diagnosis and Improvement Plan: `Digitalization_Sports_Working_Model.mdl`

*Prepared 2026-10-30. Static review only.*

**How this review was done.** I read every equation, unit, and table in the working model (116 variables), both from the raw `.mdl` file and through the Vensim MCP parser. No simulation, unit check, or test was run. Where the report states what happens "at t = 2000", the numbers are **hand-calculated from the equations at INITIAL TIME**, not simulated output. Treat behavioral consequences beyond t = 2000 as predictions to verify when you do run the model.

The review judges the model on its own terms: its variable names, comments, and equations. It does not compare against the earlier write-up or CLD.

Severity labels:

- **Critical**: will distort or break the simulated behavior on its own (sign flips, division by zero, collapse to zero, runaway initial transients).
- **Major**: decision rule or structure is behaviorally questionable, or a stock violates a physical/logical bound.
- **Minor**: units, naming, documentation, cosmetic table issues.

---

## 1. Executive summary

The model already contains the right *kind* of core mechanism: a pace → stimulation → adaptive entertainment threshold → audience interest → commercial pressure → rules → pace chain, built with standard SD molecules (asymmetric adaptive threshold, TREND function, first-order perception delays). That backbone is sound.

But several formulations will dominate the behavior before the theory gets a chance to show itself. In order of importance:

1. **The model does not start in equilibrium.** Six stocks are far from balance at t = 2000. Audience Interest, Participation, Game Analytics, Player Contagion, Game Tactics, and Physical Endurance all move strongly at t = 0 for reasons that are artifacts of initial values, not mechanism. Any early behavior you see will mostly be start-up transient. (§3)
2. **Commercial pressure maps to rules the wrong way at the extremes.** When audience interest grows *as expected*, felt pressure goes to zero, the rule level goes to zero, and desired game pace goes to **zero**. "No pressure" should mean "leave the rules alone" (REL = 1), not "stop the game." (§4.2)
3. **Game Tactics can cross zero and flip the sign of the pace adjustment.** Time to Change GP = Normal / (GT / Initial GT). GT starts decaying toward 0 immediately, and turns negative whenever the pace gap is negative. That makes adjustment time pass through infinity and then become negative, so pace moves *away* from its goal. (§4.3)
4. **Player Contagion gates the speed of pace change instead of shifting its goal**, and at the initial value it slows pace adjustment roughly 200-fold. It is also not a contagion structure. (§4.4)
5. **Audience Interest is labeled a proportion of the population but has no upper bound.** Its implied equilibrium at t = 2000 is about 2.75, i.e., 275% of the population. (§4.6)
6. **The participation hazard uses an activity frequency as an adoption rate and counts the sports hazard twice.** The "/365" divisor is a fudge factor that hides this. (§4.7)

Section 6 gives a phased plan with replacement equations. Section 7 lists the tests to run once you are ready to simulate.

---

## 2. Model map as currently implemented

### 2.1 Sectors

| Sector | Stocks | Exogenous drivers |
|---|---|---|
| Game pace | Game Pace GP, Game Output GO | — |
| Team tactics | Game Pressure GPr, Game Tactics GT, Game Analytics GA | Data Analytics Development (logistic in time) |
| Physical capacity | Physical Endurance PE, Physical Therapy PT | (via Data Analytics Development) |
| Players & social media | Player Contagion PC | Social Media Panetration (exponential in time) |
| Audience | Entertainment Threshold ET, Audience Interests AI | — |
| Commercial / rules | Historical Audience Interest HAI, Felt Commerical Pressure FCP, Rule Encourgement Level REL | — |
| Participation | Proportion of Public Sports Participation PSSP | E-sports hazard rate (logistic in time) |

All three "digitalization" forces (data analytics, social media, e-sports) are exogenous time functions. That is a legitimate design: the model explains how sport *responds* to digitalization. But it means none of the digitalization drivers can be reinforced by the sport itself. Make that choice explicit in the paper.

### 2.2 Feedback loops actually closed in the equations

| # | Loop path (as coded) | Polarity | Comment |
|---|---|---|---|
| L1 | GP → GO → Effective Sports Stimulation → Thrill effect → AI → Perceived AI → Fractional Trend → (−) Commercial Pressure → FCP → REL → Desired GP → GP | **B** | "Satisfaction" loop: faster game raises interest, which relieves pressure. Only active while Sports Stimulation / ET < 1 because the thrill table is flat above 1. |
| L2 | GP → GO → Effective Stimulation → ET → (−) Thrill effect → AI → … → (−) CP → … → GP | **R** | "Threshold ratchet": faster game raises expectations, which lowers interest, which raises pressure for even faster play. The core mechanism. It is present and correctly signed. |
| L3 | GP → Desired GP (= GP × REL) → Change in GP → GP | **R** if REL > 1 | Floating goal. Any REL > 1 produces exponential pace growth at rate (REL − 1)/Time to Change GP until the PE cap binds. |
| L4 | GP → Perceived GP → (−) GP Gap → GPr → GT → (−) Time to Change GP → GP | Conditional | A gain-modulating loop. Its polarity depends on the sign of the gap. |
| L5 | GP → GO → GA → Indicated GT → GT → Time to Change GP → GP | Conditional | Analytics only matter multiplied by GPr. When GPr = 0, analytics have no effect at all. |
| L6 | AI → (−) Fraction of SM-pursuing players → PC → Effect on speed of GP change → GP → … → AI | Conditional | Polarity depends on direction of the pace gap. |
| L7 | PSSP → Normalized ratio → Effect of PSPR → Hazard → Growth → PSSP | **R** | Social-norm loop. The table is convex, so it strengthens as participation rises. |
| — | Physical endurance | none | PE caps Desired GP via MIN, but nothing feeds back from pace to PE. The physical limit is a hard constraint, not a balancing loop. |
| — | First-order loops | B | ET adaptation, AI erosion, GA waning, PE waning, PSSP decline, and the smoothing stocks. |

**Insight about L1 and L2 together.** Because the thrill effect depends on the *ratio* Sports Stimulation / ET, and ET eventually catches up to Sports Stimulation + Social Media, the long-run ratio is approximately GO / (GO + SM). A higher pace only helps interest *transiently* (until ET adapts), and in the long run only by diluting the social media share. This is a substantive theoretical implication of your formulation, and it is a good one. Make sure it is the one you intend (see §4.5).

---

## 3. Initial-condition audit (Critical)

Hand calculation at t = 2000 with the current parameters:

| Stock | Initial value | Net flow at t = 2000 | Implied equilibrium at t = 2000 values | Problem |
|---|---|---|---|---|
| Game Pace GP | 1 | 0 | 1 | OK at t0 |
| Game Output GO | 10 | 0 | — | Redundant stock (§4.1) |
| Game Analytics GA | 50 | +47.8 /yr | ≈ 1,000 (rising to ≈ 2,000 as DA → 2) | Initial value omits the 20-year outdating time |
| Game Tactics GT | 1 | −2 /yr (indicated GT = 0) | 0 | Decays to ~0 within 2 years (§4.3) |
| Game Pressure GPr | 0 | 0 | 0 | OK at t0 |
| Physical Endurance PE | 1.5 | −0.075 /yr | 1.2 | PE settles at 80% of its "maximum" |
| Physical Therapy PT | 1 | +0.01 /yr | none | No outflow or ceiling; grows forever |
| Player Contagion PC | 5 people | ≈ +352 /yr | ≈ 357 people | Jumps 70-fold in ~1–2 years |
| Participation PSSP | 0.20 | ≈ +0.082 /yr | very high, near saturation | Inflow is 9× the outflow |
| Audience Interests AI | 0.20 | ≈ +0.085 /yr | ≈ 2.75 | Exceeds 1 although it is a proportion |
| Entertainment Threshold ET | 10 | +0.25 /yr | 10.5 | Minor: initialized to sports stimulation only, but its goal includes social media |
| HAI, FCP, REL | 0.2, 1, 1 | 0 | — | OK at t0, but driven off immediately by AI's transient |

Key arithmetic for the two largest problems:

```
Participation inflow  = (1 - 0.2) × [52 × 52/(52 + 12.59)] / 365 = 0.0918 /yr
Participation outflow = 0.2 / 20                                = 0.0100 /yr
AI inflow             = 0.0918 × Effect(1) × 1                  = 0.0918 /yr
AI outflow            = 0.2 / (1 × 30)                          = 0.0067 /yr  → AI* ≈ 0.0918 × 30 ≈ 2.75
```

**Why this matters for the theory, not just tidiness.** Because AI starts growing at roughly 40% per year, the Fractional Trend of AI immediately exceeds the 0.5%/yr expectation. Indicated Commercial Pressure is then clipped to 0, FCP and REL fall toward 0, and Desired GP falls below current pace. That negative gap then drives GPr and GT negative (§4.3). In other words, the first decade of any run is likely to be dominated by an initialization artifact that pushes pace *down*. That is the opposite of the mechanism you are trying to show.

**Principle to apply:** every stock should be initialized so that the model starts in equilibrium with drivers held at their t0 values. Then, any movement you see comes from the exogenous drivers and the feedback structure. The standard way is to write the initial value as "inflow × residence time" or to calibrate a base rate from the desired initial stock (equations in §6).

---

## 4. Sector-by-sector diagnosis

### 4.1 Game Pace and Game Output

**(Major) Game Output GO is a stock that copies another stock.**
`Change in Game Output = Change in GP × Game Output Per Game Pace` means GO ≡ 10 × GP forever. It adds no dynamics and can drift numerically. It also suggests a delay that does not exist.
*Fix:* make it an auxiliary: `Game Output GO = Game Pace GP * Game Output Per Game Pace`.

**(Major) The pace goal floats on current pace.**
`Desired GP = MIN(GP × REL, PE)` sets the goal as a multiple of where pace already is. With Change in GP = (Desired − GP)/T, this is equivalent to a fractional growth rate of (REL − 1)/T. Two consequences follow:

- Any sustained REL > 1, however small, produces exponential growth until the PE cap binds. Any REL < 1 produces exponential decay toward zero. Pace has no equilibrium other than REL = 1 or the cap.
- Every multiplier you later put on this goal (players, tactics) will behave the same way, as a growth rate rather than a level shift.

A floating goal can be a legitimate behavioral rule ("leagues always want a bit faster than now"). But if that is the intent, write it as an explicit fractional growth rate so readers see it. A more defensible alternative is anchoring on a slowly-moving **norm** of pace, e.g. the pace fans and leagues are used to over the last ~10 years. That is the standard anchor-and-adjust rule with a floating anchor (Sterman, *Business Dynamics*, ch. 13 and 15) [guess: chapter references from training knowledge].

**(Major) Physical endurance is a hard MIN cap.**
The MIN creates a kink: pace adjusts at full speed until it hits the cap, then stops abruptly. Real limits bite progressively, through fatigue, injuries, and coaches pacing players. Use a smooth constraint (an effect table of PE headroom on the goal or on the adjustment), or at least a soft-min.

### 4.2 Commercial pressure and rules (Critical)

Current chain:

```
Indicated CP = MAX(0, (EGI − FTAI)/EGI) × Base CP          EGI = 0.005 /yr
FCP          = asymmetric smooth of Indicated CP          (up 1 yr, down 3 yr)
Indicated REL = FCP × (1/CP)
REL          = smooth of Indicated REL                    (2 yr)
Desired GP   = MIN(GP × REL, PE)
```

Problems:

1. **The neutral point is misplaced.** REL = 1 (no push) occurs when interest is *flat* (FTAI = 0). When interest grows *exactly as expected* (FTAI = EGI), pressure is 0, REL → 0, and Desired GP → 0. "Meeting expectations" should be neutral or mildly relaxing, not "set the game pace to zero." This is an extreme-condition failure.
2. **The gain is extreme because EGI is in the denominator.** A 5%/yr decline in interest gives Indicated CP = (0.005 + 0.05)/0.005 = 11, so REL heads toward 11 and the desired pace toward 11× current. A decision rule that turns a modest audience decline into an 11-fold pace target is not credible as boundedly-rational league behavior.
3. **REL is unbounded and linear in pressure.** Real rule changes are incremental, bounded, and saturating. The response of rules to pressure should be a bounded effect table with value 1 at zero pressure.
4. **There is no rule path that slows the game.** Pressure is clipped at 0, so the best case is "no change." If you want the backlash or aesthetic-fatigue possibility to exist in the model, the rule response needs to be able to go below 1 when interest is growing strongly, or when the complaint signal is high.

What is good here: the TREND structure (Perceived AI, HAI, Fractional Trend) is the textbook formulation, and the asymmetric pressure adjustment (felt quickly, forgotten slowly) is behaviorally sensible.

### 4.3 Game pressure, tactics, and analytics (Critical)

```
GP Gap        = Desired GP − Perceived GP
GPr           = smooth(GP Gap, 0.5 yr)
Indicated GT  = GPr × GA / Initial DA Development
GT            = smooth(Indicated GT, 0.5 yr),  Initial GT = Initial GPr + 1
Time to Change GP = 3 / (GT / Initial GT)
```

1. **Initialization contradiction.** Initial GPr = 0, so Indicated GT = 0, while GT starts at 1. GT decays as e^(−2t): it falls to about 2% of its starting value after 2 years. Time to Change GP then rises to about 160 years, which freezes the game pace.
2. **Sign flip.** Whenever Desired GP < Perceived GP, GPr < 0 and GT < 0. Then Time to Change GP < 0, and `(Desired − GP)/negative` drives pace *away* from its goal. As GT crosses zero, the adjustment time passes through ±∞ (division by zero exactly at 0). Given §4.2, this is likely to occur early in a run.
3. **Magnitude.** GA is not normalized by its own initial value (50) but by Initial Data Analytics Development (1), and GA grows toward about 1,000. Indicated GT can therefore reach hundreds of times GPr. That makes Time to Change GP potentially shorter than the 0.0625-year time step, producing numerical instability.
4. **Multiplicative gate.** Because Indicated GT = GPr × GA, analytics have zero effect when pressure is zero. That contradicts the idea that analytics alone push teams toward more efficient, faster styles.
5. **Two smoothings of the same signal.** GPr is a smooth of the gap and GT is a smooth of GPr × GA. "Game pressure" is then just a delayed copy of the rule-induced gap. If game pressure is meant to be *competitive* pressure among teams (matching faster opponents), it needs its own driver. In an aggregate one-league model, a reasonable proxy is the perceived *trend* in league pace: teams feel pressure when the league is speeding up.
6. **Where does tactics act?** Tactics currently change only *how fast* pace approaches a goal set by rules and players. If analytics-driven tactics make the game faster in their own right, as in the three-point revolution, tactics should shift the **goal**, the level of pace teams aim for. Decide which you mean. My recommendation is the goal, with an optional secondary effect on adjustment time.
7. **Units.** GT and GPr are measured in "GP" units. A tactics index measured in game-pace units is hard to interpret. `Initial GT = Initial GPr + 1` adds a bare number to a GP quantity, which is a hidden parameter.

### 4.4 Players and social media (Major)

```
PoE = SMOOTH(Social Media Panetration, Time to Perceive Trend)
Indicated fraction = PoE / (AI + PoE)
PC  → (Indicated fraction × 500 − PC)/1 yr
Effect of PC on GP Change = Table(PC/500),   table(0) = 0
Change in GP = (Desired GP − GP)/Time to Change GP × Effect of PC
```

1. **Effect on speed, not on goal.** The effect multiplies the *rate* of pace adjustment. At the initial PC/Total = 0.01, the table returns about 0.005. Pace adjustment is therefore about 200 times slower than the normal 3 years. At PC = 0 the game pace cannot change at all, even if rules change. Behaviorally, players chasing exposure should change **how they play** (the pace they aim for), not the league's ability to adjust. Every effect multiplier should also equal 1 at the reference condition.
2. **Not a contagion structure.** The name says contagion, but the formulation is first-order goal-seeking toward an exogenous-driven target. There is no imitation term (adopters influencing non-adopters). A contagion would use the standard diffusion structure: adoption ∝ non-adopters × (autonomous adoption + contacts × adopter fraction). That is the Bass or SI molecule.
3. **Scale-mismatched attention share.** PoE (0.5, in "Interest") is compared with AI (0.2, in "Interest"). The same social media variable is compared with Sports Stimulation (≈10) in the threshold sector. The two "Interest" quantities are on different, arbitrary scales, so the 71% initial SM-pursuing share is a scaling coincidence rather than a meaningful number.
4. **Questionable sign.** When audience interest in the sport falls, more players pursue social media exposure. That may be intended (players compensate personally when the league is less popular), but it needs a stated justification. The more direct driver is the *payoff* to exposure, i.e., social media reach × volume of clip-worthy events. That would also close a loop from pace back to players.
5. **Naming.** PoE smooths with "Time to Perceive Trend", but it smooths a level, not a trend. Rename it "Time to Perceive Social Media Reach".

### 4.5 Stimulation and entertainment threshold

The structure is the strongest part of the model. ET is an asymmetric adaptive expectation. It rises toward experienced stimulation in 2 years and relaxes in 5 years, which is a hedonic-adaptation ratchet. Keep it.

Issues:

1. **(Minor) Inconsistent initialization.** ET is initialized to Sports Stimulation only (10), while its goal is Sports + Social Media (10.5). Initialize it to `INITIAL(Effective Stimulation)` or normalize the ratio (below).
2. **(Major) Thrill only hurts.** The thrill table is flat at 1 for ratio ≥ 1. A game that exceeds expectations gives no extra interest. That removes the "early acceleration excites fans" phase. If you believe novelty attracts, extend the table above 1 with diminishing returns (a log-like shape is consistent with Weber–Fechner reasoning).
3. **(Major) Scale.** Social media enters additively at 0.5 against sports stimulation of 10. Its effect on the threshold is therefore at most ~15% by 2040, and that size depends entirely on the arbitrary constants Game Output Per Game Pace = 10 and Unit Stimulation = 1. Convert both to **indices equal to 1 at t0**, and give social media an explicit weight parameter you can vary in sensitivity analysis.
4. **(Theory choice) Additive vs. multiplicative social media effect.**
   - *Additive* (current): Threshold goal = Sports + w × SM. A faster pace can partially outrun social media, so leagues can "keep up".
   - *Multiplicative*: Threshold goal = Sports × SM^e. The threshold moves proportionally with the game, so pace can never outrun expectations. This is a pure treadmill.

   These yield different stories. The additive form preserves the possibility that pace growth is a rational response. I recommend keeping it, with indices and a normalized ratio.
5. **(Major) Social media grows exponentially without limit.** Penetration is bounded in reality. Use a logistic like the other two drivers.

### 4.6 Audience interest (Major)

1. **No upper bound.** AI is documented as "proportion of population watching sports" but its inflow has no (1 − AI) term. Its implied equilibrium at t0 is about 2.75.
2. **Inflow is borrowed from the participation inflow.** Interest Accumulation = Growth in PS Participation × Thrill. Only people *starting* to participate can become viewers, and viewing recruitment then inherits the participation formulation's problems (§4.7). A cleaner rule: viewers are recruited from non-viewers at a base rate, modulated by thrill and by participation as a supporting effect.
3. **Viewer proportion initialized to the participation proportion.** AI0 = PSSP0 = 0.2 conflates watchers and participants. Use a separate, data-based initial value.
4. **Same effect used twice.** The thrill effect multiplies both the inflow and the "years enjoying" in the outflow, with the same table. Two mechanisms can be justified (fewer new fans, faster loss of existing fans), but give them separate tables and document why.
5. **(Critical, conditional) Division by zero.** If Sports Stimulation → 0, which can follow from §4.2, then the thrill effect → 0, Years Enjoying Watching Sports → 0, and Interest Erosion = AI / 0. Give the retention effect a floor > 0.

### 4.7 Public sports participation and e-sports (Major)

1. **Activity frequency used as adoption hazard.** "Base Hazard Rate = 52 /yr" is documented as "participate once a week". That is how often *participants* play, not the rate at which *non-participants become participants*. Dividing by "Total Potential Activity = 365" (units "1") rescales it to look like an adoption probability, but the conversion has no behavioral meaning.
2. **Double counting.** Actual Hazard = H × H/(H + E). The sports hazard H enters once directly and again inside "Relative Attractiveness". The attractiveness share should be computed from *attractiveness* variables, then multiply a base adoption rate once.
3. **Convex social-norm table.** The points (0,0), (1,1), (2,2), (5,7) have slope that *increases* with participation, producing a runaway reinforcing loop. Social-norm effects usually saturate: use an S-shaped or concave table.
4. **E-sports documentation mismatch.** The initial-rate comment says it is "kept so the initial state is unchanged", but the logistic gives 12.59 at t = 2000, not 12. The maximum's comment says "weekly, parity with base PSP frequency", but the value is 100 /yr, not 52.
5. **E-sports affects only adoption, never quitting or viewing.** Note whether that is intended. E-sports competing for *viewing time* would arguably act on AI directly.

### 4.8 Physical endurance and physical therapy (Major)

1. **"Waning" of a capacity.** PE drains at 1/20 per year in proportion to its own level, while improvement closes the gap to a maximum in 5 years. In balance, PE sits at 80% of its maximum. It is unclear what process "waning" represents if it is not driven by load.
2. **No load feedback.** Nothing connects pace to endurance. If the physical limit is meant to be a *balancing loop* (faster play → more fatigue and injury → lower sustainable pace), waning or the indicated capacity must depend on load (pace relative to capacity).
3. **Double use of one effect.** "Effect of PT on PE Waning" both slows waning and raises the maximum. The name covers only one use. Split it, or pick one channel.
4. **PT grows without limit.** There is no outflow or ceiling, so the endurance cap keeps rising linearly forever. Add diminishing returns.

### 4.9 Data analytics

1. **(Minor) Naming.** "DA Increase Ceiling" is the *increment* (the level goes from 1 to 2), not a ceiling. Rename it "DA Increase Range".
2. **(Minor) Pseudo-delay.** "GA Delivery Time" divides an inflow without any stock in transit. It is a conversion constant in disguise; fold it into the productivity parameter.
3. **(Major) Units.** GA is measured in "DA Level", the same unit as the exogenous index, although it is an accumulated stock of analytics knowledge. Give it its own unit.

---

## 5. Units, naming, and documentation (Minor, but do it once)

- **Spelling** (renaming in Vensim updates all references): Commerical → Commercial, Preceived → Perceived, Encourgement → Encouragement, Panetration → Penetration, Thrillness → Thrill, Adaption → Adaptation. The participation acronym is "PSSP" in the stock name but "PSP" elsewhere.
- **"Interest" unit is overloaded.** It is used for stimulation, social media, the threshold, and viewer share. Split it into Stimulation (index) and Viewer Share (Dmnl).
- **Hidden numerical literals:** the `+ 1` in Initial GT, and `365` with units "1".
- **Table ranges:** the thrill table declares x in [0, 1] but has a point at 2. The PSPR table declares x in [0, 10] but its last point is 5.
- **Normalizations:** normalize by a variable's own `INITIAL()` value rather than by a separate constant, which goes out of sync when you change initial values. For example, GA is divided by Initial Data Analytics Development rather than by initial GA.
- **Documentation:** almost no constant has a comment or source. Every time constant and table should state its meaning, its unit, and whether it is a placeholder.
- **Time step:** 0.0625 is fine for the current 0.5-year constants. Once Time to Change GP can shrink (§4.3), keep every effective time constant at least 4–5 × DT, which bounded tables ensure.

---

## 6. Improvement plan

Each phase leaves the model runnable. Do the phases in order, because Phase 1 removes artifacts that would otherwise hide whether Phase 2 changes are working. The equations below are proposals in Vensim syntax. Placeholder table values are marked and need your judgment or evidence.

### Phase 0 — Housekeeping (½ day)

1. Rename variables per §5. Add units and comments to every constant, marking placeholders.
2. Introduce index units: Stimulation Index, Analytics Index, Tactics Index (all Dmnl, 1 at t0).
3. Turn Game Output into an auxiliary:
   ```
   Game Output GO = Game Pace GP * Game Output Per Game Pace     ~ GO
   ```
   Delete `Change in Game Output`.

### Phase 1 — Structural fixes that remove artifacts (Critical items)

**1a. Initialize every stock in equilibrium.**

```
Game Analytics GA = INTEG(Increase in Game Analytics - Waning of Data Information,
                          Increase in Game Analytics * Time for Data to be Outdated)
Increase in Game Analytics = Game Output GO * Data Analytics Development * Analytics Generated per Output
      ~ Analytics/Year          (Analytics Generated per Output ~ Analytics/(GO*DA Level*Year))

Entertainment Threshold ET = INTEG(Change in Entertainment Threshold, INITIAL(Effective Stimulation))

Physical Endurance PE = INTEG(Change in PE, Indicated Sustainable PE)        (see 2e)
```

For participation and audience interest, calibrate the base rate from the desired initial stock (2c, 2d).

**1b. Commercial pressure → rules: neutral at expectation, bounded, able to relax.**

```
Interest Shortfall = Expected Growth of Interest - Fractional Trend of AI            ~ 1/Year
Indicated Commercial Pressure = Interest Shortfall / Reference Interest Shortfall   ~ Dmnl
Reference Interest Shortfall = 0.01                                                  ~ 1/Year  [placeholder]
Felt Commercial Pressure FCP = INTEG(Change in FCP, Indicated Commercial Pressure)  (keep asymmetric smoothing)
Indicated REL = Table for Effect of Pressure on Rules(Felt Commercial Pressure FCP) ~ Dmnl
Table for Effect of Pressure on Rules([(-3,0.9)-(5,1.2)],
      (-3,0.95),(-1,0.98),(0,1),(1,1.03),(2,1.06),(3,1.08),(5,1.1))        [placeholder shape]
```

- FCP can now be negative, meaning interest is beating expectations. The table gives REL slightly below 1, a mild slowing or backlash channel. Delete the negative side if you do not want that.
- For the **equilibrium test**, set Expected Growth of Interest = 0 so the model starts at rest. Then restore it as a scenario.

**1c. Tactics: bounded index acting on the goal; pressure from competitive trend.**

```
Perceived League Pace Trend = TREND(Game Pace GP, Time to Perceive Pace Trend, 0)   ~ 1/Year
Game Pressure GPr = SMOOTH(MAX(0, Perceived League Pace Trend) / Reference Pace Trend, Time to Change GPr)   ~ Dmnl
Relative Game Analytics = Game Analytics GA / INITIAL(Game Analytics GA)            ~ Dmnl
Indicated GT = Effect of Analytics on Tactics(Relative Game Analytics) * Effect of Pressure on Tactics(GPr)
Game Tactics GT = INTEG((Indicated GT - Game Tactics GT)/Time to Change GT, 1)     ~ Dmnl
Effect of GT on Desired GP = Table(GT)   with (1,1), increasing, saturating   [e.g. (0.5,0.95),(1,1),(2,1.08),(4,1.12)]
```

- Both effect tables pass through (1,1) and are bounded.
- If you also want tactics to speed up adjustment, keep `Time to Change GP = Normal Time to Change GP / Effect of GT on Adjustment(GT)` with a table bounded in, say, [0.5, 3]. That avoids the zero-crossing.

**1d. Pace goal with a slowly-floating norm and bounded multipliers.**

```
Pace Norm = SMOOTH(Game Pace GP, Time to Establish Pace Norm)       Time to Establish Pace Norm = 10 yr [placeholder]
Indicated GP = Pace Norm * Rule Encouragement Level REL
               * Effect of GT on Desired GP * Effect of Players on Desired GP
Desired GP = Indicated GP * Effect of Physical Headroom(Physical Endurance PE / Indicated GP)
Effect of Physical Headroom: (0,0),(0.8,0.8),(1,0.95),(1.2,1),(2,1)   [smooth cap; placeholder]
Change in GP = (Desired GP - Game Pace GP) / Time to Change GP
```

Remove `Effect of PC on GP Change` from Change in GP. Players now act on the goal (2a).

**1e. Division-by-zero guards.** Once the tables above are bounded away from zero, guards are rarely needed. Still give retention a floor:

```
Years Enjoying Watching Sports = Normal Years Enjoying Watching Sports * Effect of Thrill on Retention
Effect of Thrill on Retention: table with minimum ≥ 0.2   [placeholder]
```

### Phase 2 — Behavioral reformulations (Major items)

**2a. Player contagion as diffusion, driven by the payoff to exposure.**

```
Players Not Pursuing SM = Total Players - Players Pursuing SM PC
Exposure Payoff Index = Social Media Index * (Game Output GO / INITIAL(Game Output GO))   ~ Dmnl
SM Adoption = Players Not Pursuing SM * Exposure Payoff Index
              * (Autonomous Adoption Fraction + Contact Rate * Adoption Fraction * PC / Total Players)
SM Abandonment = Players Pursuing SM PC / Average Duration of SM Pursuit
Players Pursuing SM PC = INTEG(SM Adoption - SM Abandonment, Initial Pro SM Players)
Effect of Players on Desired GP = Table(PC / Total Players / INITIAL(PC / Total Players))   through (1,1), bounded
```

- Calibrate Autonomous Adoption Fraction so the stock starts in balance.
- The GO term closes a new loop: faster game → more clip-worthy events → bigger payoff to exposure → more players chasing it → faster game. That loop is reinforcing.
- Keep or drop the audience-interest term from the old formulation depending on whether you can justify it (§4.4, point 4).

**2b. Social media as a bounded, normalized driver.**

```
Social Media Index = 1 + (Max SM Index - 1) * (LOGISTIC(Time) - LOGISTIC(INITIAL TIME)) / (1 - LOGISTIC(INITIAL TIME))
      where LOGISTIC(t) = 1 / (1 + EXP(-(t - SM Midpoint)/SM Steepness))
Sports Stimulation Index = Game Output GO / INITIAL(Game Output GO)
Effective Stimulation = Sports Stimulation Index + SM Weight * (Social Media Index - 1)
Thrill Ratio = (Sports Stimulation Index / Entertainment Threshold ET)
Effect of Thrill on Recruitment = Table(Thrill Ratio):  (0,0.1),(0.5,0.5),(0.8,0.85),(1,1),(1.5,1.12),(2,1.18)   [placeholder]
```

In Vensim, write the logistic out in full, since there is no user-defined function. Apply the same "starts exactly at its initial value" normalization to the e-sports driver to fix the 12 vs. 12.59 mismatch.

**2c. Audience interest as a bounded viewer share.**

```
Viewer Share AI = INTEG(Viewer Recruitment - Viewer Loss, Initial Viewer Share)    ~ Dmnl
Viewer Recruitment = (1 - Viewer Share AI) * Base Viewer Recruitment Rate
                     * Effect of Thrill on Recruitment * Effect of Participation on Recruitment
Viewer Loss = Viewer Share AI / Years Enjoying Watching Sports
Base Viewer Recruitment Rate = Initial Viewer Share / (Normal Years Enjoying Watching Sports * (1 - Initial Viewer Share))
Effect of Participation on Recruitment = Table(PSSP / INITIAL(PSSP)) through (1,1)
```

Optionally add an e-sports attractiveness term to recruitment or loss, if you want e-sports to compete for viewing directly.

**2d. Participation as an adoption process with a proper attractiveness share.**

```
Growth in PS Participation = (1 - PSSP) * Base Adoption Rate * Effect of Norm on Adoption(PSSP/INITIAL(PSSP))
                             * Relative Attractiveness of PSP
Relative Attractiveness of PSP = [Attr PSP / (Attr PSP + Attr Esports)] / INITIAL([same])      ~ Dmnl, 1 at t0
Base Adoption Rate = INITIAL(PSSP) / (Years Enjoying Public Sports * (1 - INITIAL(PSSP)))     ~ 1/Year
Effect of Norm on Adoption: concave/S-shaped through (1,1), e.g. (0,0.6),(1,1),(2,1.25),(4,1.4)   [placeholder]
```

In Vensim, define the initial share as its own `INITIAL()` auxiliary. Delete Total Potential Activity and the 52/yr frequency, or keep 52/yr only as a documented activity frequency if you need it elsewhere.

**2e. Physical capacity with load feedback.**

```
Potential Capacity = Base PE * Effect of PT on Capacity(PT / INITIAL(PT))
Load Ratio = Game Pace GP / Potential Capacity
Indicated Sustainable PE = Potential Capacity * Effect of Load on Capacity(Load Ratio)
      Effect of Load on Capacity: (0,1),(0.8,1),(1,0.95),(1.2,0.8),(1.5,0.6)   [placeholder]
Change in PE = (Indicated Sustainable PE - Physical Endurance PE) / Time to Build PE
PT Improvement = Base PT Improvement * Data Analytics Development * PT Improvement per DA
                 * MAX(0, 1 - Physical Therapy PT / Maximum PT Level)
```

The load ratio uses *potential* capacity, not PE, in the denominator. This avoids a spurious reinforcing collapse loop (PE down → load up → PE further down). The loop added is the intended balancing "physical limit" loop: faster game → higher load → lower sustainable capacity → lower ceiling on pace.

### Phase 3 — Documentation of the dynamic hypothesis

After Phases 1–2, redo the loop table in §2.2 for the revised model and name each loop. For each loop, write one sentence of the behavioral rationale and the evidence you have for each link. That table becomes the backbone of the model-description section of the paper.

---

## 7. Tests to run later (not run now)

When you are ready to simulate, in this order:

1. **Units check** in Vensim (Model > Units Check) after Phase 0.
2. **Equilibrium test.** Freeze all three exogenous drivers at their t0 values and set Expected Growth of Interest = 0. Every stock should stay flat. Any stock that moves points to a remaining initialization error.
3. **Step tests, one driver at a time.** Step social media, then e-sports, then data analytics. Check the direction and timing of pace, interest, and pressure against your hypotheses.
4. **Extreme-condition tests.**
   - Social media index = 1 forever.
   - E-sports attractiveness = 0, then very large.
   - Physical capacity very low: pace must stay below it smoothly.
   - Expected interest growth very high and very low: REL must stay within table bounds.
   - Total players all pursuing social media.
   - Game pace initial value near 0.

   Nothing should go negative, divide by zero, or exceed a logical bound.
5. **Loop knockout.** Disable one loop at a time (e.g., fix ET constant, fix REL = 1, fix PC) to see which loop drives each phase of behavior.
6. **Time-step test.** Halve TIME STEP. Results should not change materially.
7. **Sensitivity.** Run Monte Carlo over all placeholder parameters and table shapes, especially SM Weight, the pressure→rules table, the thrill table, and threshold adaptation and relaxation times.
8. **Reference modes.** Compare pace trajectories with the empirical series you have collected, at least qualitatively.

---

## 8. Decisions that are yours to make

These choices change the theory, not just the code. I gave a default in the plan, but you and Tim should confirm them.

1. **Pace goal.** Floating on current pace (pure growth-rate logic), or anchored on a slowly-moving pace norm (recommended)?
2. **Tactics.** Does analytics-driven tactics change the *level* of pace teams aim for (recommended), the *speed* of adjustment, or both?
3. **Social media and the threshold.** Additive, so pace can outrun social media (recommended), or multiplicative, a pure treadmill where pace never catches up?
4. **Backlash.** Should rules be able to slow the game when interest is growing or complaints are high? If yes, keep the negative side of the pressure→rules table. A fuller "aesthetic fatigue" mechanism would need a separate variable (e.g., perceived stylistic homogeneity) feeding audience interest. That is a larger extension to scope separately.
5. **Players' motive.** Is pursuit of exposure driven by social media payoff × salient events (recommended, closes a loop), or by the sport's audience interest falling (current)?
6. **E-sports.** Should it compete for participation only (current), for viewing time, or both?
7. **Physical limit.** Hard ceiling (current) or load-driven balancing loop (recommended)?
