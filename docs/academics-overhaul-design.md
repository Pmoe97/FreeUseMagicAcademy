# FUMA Academics Overhaul — Design Plan

## Context

Free-Use Magic Academy (`main.html`, ~19.7k lines / 3.8MB single-file Perchance generator + `main.pjs` config) was ported from a prior corporate-sim project (FUOC) with a reskin but not a structural rework. The result: the game's central system — what should be "running a school" — is mechanically still "running a business":

- **Academy tab** (`academyTab`): a pure idle-clicker. "Magical classes" are Cookie-Clicker-style production buildings you buy/upgrade with a "Bulk Buy" multiplier, passively generating gold/sec across multiple "location" subtabs (old business branches).
- **Upgrades tab** ("Arcane Power"): capital-investment upgrades + a full prestige/reset loop ("Graduate Now" wipes gold/classes/progress, keeps Influence Points, Alumni, and Permanent Holdings).
- **Tuition tab**: literal old payroll — weekly "stipends" paid *to* students, auto-pay/late-penalty mechanics, an "Accountant Coverage" candidate pool, and a full bank-loan system (Gringotts Arcane™, bonds, compounding interest).

None of this reflects "headmaster manages a school." The goal of this plan is to fully replace these three tabs with a **scheduled school-management sim**: hire and develop teachers, build a real class catalog, run a weekly period-by-period schedule that students and staff actually attend, resolve grades from a genuine (gradually-influenceable) formula, build reputation against real rival schools, and handle discipline — including adult-themed misconduct — through a combined meter + interactive-incident system. This is explicitly *not* a clicker anymore; the loop is closer to a management sim with Persona-style "every period is a place you can visit" texture.

This plan is the result of extensive clarification with the user (5 rounds of questions) and is meant to be saved as a permanent design reference in the repo, followed by a phased implementation.

**Update (post-implementation-start):** the user intentionally runs multiple parallel Claude Code sessions against this repo. A separate concurrent session (PRs #1–#4, "harry-potter-creative-pass"/"harry-potter-game-overhaul") independently built a real House system, retitled the student career ladder away from corporate language, and did a content-cleanup pass on remaining business-tier names — all before any of that was reflected in this doc. The Houses/Dueling sections below (System 6/7) have been reconciled against what actually exists; see the "Existing implementation" notes inline. Future sessions picking up this doc should `git fetch`/check `origin/master` for drift before assuming the doc is current.

## Design Pillars

1. **Every period is a place.** Classes, tutoring sessions, and sporting events are all "visitable" — the Headmaster (and other characters) can attend live for an interactive scene, or let it auto-resolve via formula. Attendance is data that feeds back into outcomes.
2. **Formula-driven, but influenceable.** Grades, reputation, and match outcomes resolve from stat formulas (predisposition + skill + relationship + luck + effort), not clicks. Individual students are easy to move; moving the whole school is deliberately harder — this is the core "management" tension.
3. **Two currencies, one goal.** Money (tuition, donor funding) keeps the lights on; Reputation (grades, sports, faculty quality) is the actual growth/prestige metric, measured against real rival schools.
4. **No more hard reset.** Prestige is replaced by a persistent Legacy/Endowment system — reputation and achievements compound over time instead of wiping the save.
5. **Content intensity is a dial, not a wall.** Discipline/misconduct content has its own dedicated intensity control, independent of the general Atmosphere/Consent sliders — but even a "reserved" school can drift/corrupt over time, so the dial sets pace/ceiling, not a hard gate.

## Scope

**In scope (full rework):** Academy tab → Curriculum & Scheduling; Upgrades tab → Faculty Development + Legacy/Endowment; Tuition tab → School Budget (tuition income, donor/board funding, teacher payroll); Duel system → folded into Electives/Sports; People tab → reworked roster with clear Student/Faculty/Alumni separation; HR tab → lightly adjusted to match the new applicant-generation system (stays functionally the same purpose: shaping difficulty and admissions).

**Out of scope (untouched):** Social tab, Gifts tab, Groups tab (this is multi-person chat, not houses — confirmed via code read), Story tab, Headmaster's Tower chat. These are the relationship/roleplay layer and are not part of the "Business" gutting.

**Note on Houses (updated):** Houses are no longer a loose filter attribute — a concurrent session already built a real House system (see System 6's "Existing implementation" note). This plan's job now is to layer the *external* rival-school competition and curriculum/scheduling integration on top of that existing internal house system, not to build houses from scratch.

---

## System 1: Faculty (Teachers)

- **Hiring**: Reuse/upgrade the existing candidate-pool pattern (currently seen in Tuition's "Accountant Coverage" pool). Candidates generate deterministically and refresh instantly like today, but generation becomes thematically aware of the role/subject being hired for (not pure mad-lib). Full custom character creation remains available for both teachers and students.
- **Progression**: Full mechanical depth — teachers earn XP from teaching, level up subject mastery and a general "Teaching Skill," have a morale stat, and can retire out naturally over time. This gives payroll/attention real stakes.
- **Burnout (confirmed distinct state)**: Burnout is its own intermediate stage, not just a low-morale label. Low morale over time pushes a teacher into a "burnt out" state — reduced teaching quality, occasional erratic incidents (potentially feeding the Discipline system, System 8) — which is recoverable with attention (time off, a raise, a supportive conversation) or, if ignored, escalates to the teacher quitting.
- **Payroll**: Reuse/rework the existing weekly-payday UI (Auto-Pay toggle, Pay Early bonus, late-payment consequences) for teacher salaries only. Students no longer receive stipends — they pay tuition instead (see System 9).
- **Assignment**: A teacher is hired into a subject + skill level, then assigned to one or more class sections in the weekly schedule (System 3).

## System 2: Curriculum & Class Catalog

### Proposed Aptitude Stats

To give the catalog real hooks into the Grading formula (System 5), each subject is tagged with the 1–2 student aptitude stats it primarily draws on. This is a lightweight starting proposal, not a final stat system — six stats, kept broad enough to reuse across grading, discipline, and sports:

| Stat | Represents |
|---|---|
| Intellect | Book-smarts, theory, memory |
| Focus | Precision, patience, steady-hand technical work |
| Creativity | Improvisation, unconventional problem-solving |
| Willpower | Discipline, resolve, raw magical force |
| Presence | Charisma, performance, social/persuasive magic |
| Physicality | Reflexes, stamina, coordination |

### The Catalog (14 subjects)

Split into Core (foundational, mostly Freshman/Sophomore), Advanced (unlock Junior+, mostly double period), and Non-Academic Electives (6th period, optional, feed the House/Sports system).

| Subject | Category | Years | Meeting Pattern | Stats | Flavor |
|---|---|---|---|---|---|
| Charms | Core | All | MWF, single | Focus + Creativity | Everyday practical spellwork — levitation, mending, minor enchantments. |
| Potions | Core | All | TTh, single | Focus + Intellect | Precise brewing under time pressure; lab-based. |
| Herbology | Core | All | MWF, single | Focus + Willpower | Magical botany, greenhouse fieldwork. |
| History of Magic | Core | All | MWF, single | Intellect | Lecture-heavy; the most naturally auto-resolvable subject — a good candidate for the first implementation pass since it needs the least "visitable scene" scaffolding. |
| Astronomy | Core | All | TTh, single | Intellect + Focus | Occasionally runs evening sessions — a natural hook for after-hours scenes. |
| Transfiguration | Core → Advanced | All (upgrades Junior+) | TTh single (Fresh/Soph) → MW double (Junior+) | Willpower + Focus | The core technical backbone subject; gets harder and longer as students advance. |
| Alchemy | Advanced | Junior+ | MW, double | Intellect + Focus | Fusion of Potions theory and high-level transmutation. |
| Enchanting | Advanced | Junior+ | TTh, double | Creativity + Focus | Crafting and imbuing magical items. |
| Divination | Advanced | All (Graduate-only seminar tier) | MWF, single | Presence + Creativity | Fortune-telling and intuitive magic; gains a deeper seminar version for Graduates. |
| Magical Creatures | Advanced | All | TTh, single/fieldwork | Presence + Willpower | Care of and interaction with magical beasts; courage- and empathy-coded. |
| Ritual Magic | Advanced (restricted) | Senior/Graduate only | MW, double | Willpower + Intellect | Requires headmaster sign-off to enroll — a natural gate for reputation-risk flavor. |
| Combat Magic (Dark Arts Defense) | Advanced (restricted) | Junior+ | TTh, double | Physicality + Willpower | Restricted-enrollment defense/combat theory; higher scandal-risk if things go wrong in this class. |
| Dueling Club | Non-Academic Elective | All (opt-in) | 6th period TTh + rotating Saturday matches | Physicality + Willpower | Folds in the old standalone Duel minigame (System 7); individual and team dueling. |
| Skyclash | Non-Academic Elective | All (opt-in) | 6th period MWF + rotating Saturday matches | Physicality + Focus | Flagship broom-sport; the primary house-pride team sport (System 7). |

Non-Academic Electives run on a separate 6th period, outside the 20-slot academic budget below, and only students opted in attend them.

### Reconciling the Weekly Slot Math

Each student has a fixed **20-slot weekly academic budget** (4 class periods × 5 weekdays; the 5th daily period is always the rotating free period, confirmed separately, and isn't part of this budget). Meeting frequency: standard singles meet 3×/week (MWF) = 3 slots; lighter singles meet 2×/week (TTh) = 2 slots; doubles meet 2×/week but cost 2 slots per meeting = 4 slots. The model doesn't force every slot to be filled — some slack (open/study time) is expected and fine:

- **6-subject courseload** (typical Freshman/Sophomore, all standard Core singles): 6 × 3 slots = 18 of 20 filled, 2 slots left open. Matches the low end of the 6–8 subject range cleanly.
- **8-subject courseload** (typical Senior/Graduate, mixing in Advanced doubles): 6 lighter (TTh) singles (6 × 2 = 12) + 2 doubles (2 × 4 = 8) = 20 of 20 filled exactly. As courseload grows, subjects naturally shift toward the lighter TTh pattern to make room for double periods.

This is a draft for review, not final — the specific subject list and exact meeting-pattern assignments should be revisited once the Scheduling engine (System 3) is actually being built.

## System 3: Scheduling Engine

- **Week structure**: Monday–Friday, 5 periods/day. Each student attends 4 classes + 1 free period per day. A 6th period exists for electives (Dueling, Sports) — optional, not all students participate.
- **Weekly slot math (confirmed)**: A student has a 20-slot weekly budget (4 periods × 5 days). Subjects meet on a recurring pattern (MWF or TTh style) rather than daily, typically 2–3×/week. A "double period" subject occupies 2 of that day's 4 slots per meeting, so it consumes more of the weekly budget per meeting but meets less often. With 6–8 subjects/semester, this spends the 20-slot budget without requiring every subject to meet every day. This is the confirmed model the scheduling engine should implement.
- **Free period logic (confirmed)**: A student's free period *rotates day-to-day* (e.g. Monday period 3, Tuesday period 1, ...) — it is not a fixed slot for the semester. The scheduling engine must therefore vary each student's daily layout across the week, not just assign one static weekly grid per student.
- **Weekend**: Saturday/Sunday each have 1 tutoring/study period, usable by struggling students to improve grades. Saturdays additionally host rotating sporting events.
- **Enrollment**: Mostly automated with manual override. The Headmaster manages the teacher/course side (assign teacher + subject + time block, set section capacity); students auto-enroll into sections based on interest/stats/capacity, with the ability to manually reassign individual students when desired.
- **Attendance & visitability**: Every scheduled period — a class section, a tutoring session, a sporting event — is a "place." Default resolution is automatic/formula-driven (for scale: 50–100 students × 5 periods × 5 weekdays is far too much to force manual play on every instance). The player (and potentially other characters) can choose to "walk into" any specific instance for a live interactive scene; who is actually in attendance for that instance modifies the formula outcome for everyone present. Saturday sporting events follow the same pattern explicitly: attend live, or skip and let it simulate.

## System 4: Students & Lifecycle

- **Scale**: Target 50–100 concurrently enrolled students at launch, with the data model built to scale further (some students "featured"/fully detailed, especially those with active relationships; others lighter-weight/background, promotable to featured on demand).
- **Full lifecycle**: Students age up through Freshman → Sophomore → Junior → Senior → Graduate over defined in-game time. After Graduate year they leave as Alumni (existing concept, preserved from the old prestige modal) or become hireable Faculty (per System 1). New Freshman cohorts enroll each cycle to keep the population steady.
- **Money**: Students never receive money; they generate tuition revenue (System 9).

## System 5: Grading

- Grades resolve via a **complex weighted formula**: predisposition/aptitude (rolled trait) + accumulated skill + relationship level with the teacher/Headmaster + a luck/variance term + effort/attendance.
- Design intent: a single student is easy to move quickly (tutoring, attention, relationship investment); moving the *whole school's* average is deliberately slow and requires systemic investment (better teachers, facilities, curriculum quality) — this is the core management tension the player should feel.
- Weekend tutoring periods are a direct, visitable lever for struggling individual students.

## System 6: Reputation & Rival Schools

- Reputation is a first-class currency alongside Money, driven by aggregate grades, sports results, and faculty quality.
- **Real rival schools** (3–5), each with their own tracked reputation/stats and a visible ranking/comparison — not just flavor text.
- **Two competition layers**: Houses compete against each other *within* the school (internal rivalry, house standings), and the school as a whole competes against rival schools (external rivalry). Sporting events are the primary mechanical expression of both layers — some matches are house-vs-house, some are school-vs-school.
- **Houses get a full data model (confirmed)**: this is real added scope beyond simple visual grouping, explicitly confirmed by the user. No dedicated "Houses tab" is planned — this data surfaces inside People and the Reputation/Sports views.

**Existing implementation (already built by a concurrent session, PRs #1–#4 — reuse, don't rebuild):**
- `HOUSES` constant: 4 houses (Griffinmoor 🦁, Serpentyne, Ravensworth, Badgerholt), each with id/displayName/emoji/crest/mascot/motto/traits/personalityHint/colors/uniformPalette/commonRoomFlavor.
- `student.house` — every student is assigned a house (`pickHouseForNewStudent`, `assignHouseAndUniform`), with uniforms generated per-house (`generateHouseUniform`).
- `gameState.houseStandings` — a simple `{houseId: points}` map. `awardHousePoints(houseId, points, reason)` adjusts it with a notification; `updateHouseStandingsUI()` renders it into `#houseStandingsList` (the House Cup).
- House-vs-house dueling already exists: `getHouseChampion(houseId)` picks a house's strongest eligible student by stats, `runHouseDuel(houseA, houseB)` / `triggerRandomHouseDuel()` resolve a formula-driven outcome with flavor-text lines and award house points to the winner.
- A **"Prefect" naming collision** exists in the codebase: the old clicker-automation "prefect assigned to a class" system (`getPrefectBonuses`, `prefectAssigned`, `getPrefectUpgradeCost`, etc.) is untouched, while the concurrent session *also* added `isPrefect(student)` as a new in-house student social-hierarchy role (`showPrefectEnrollmentModal`, `selectPrefectCandidate`, `enrollOrUpgradePrefect`). Both meanings now coexist under the same word. Not blocking, but worth disambiguating (e.g. renaming one) before either system grows further — flagged in Open Items.

**Still to build (this plan's actual remaining scope for System 6):** the *external* rival-school layer (3–5 named rival schools, their own reputation tracking, school-vs-school competition) — nothing like `RIVAL_SCHOOLS` exists yet, confirmed via code search. The Reputation formula below should treat `gameState.houseStandings` as the data source for its internal-house-competition sub-component rather than inventing a parallel points system.

**Open design tension:** `getHouseChampion` currently picks a house's strongest *any* student by stats, with no gate on Dueling/Sports elective enrollment. This plan's System 7 originally assumed only elective-enrolled students are eligible to represent a house/school. Reconciling these (should champion-picking start respecting elective enrollment once Curriculum/Scheduling exists, or is "any strong student can be tapped" the intended feel?) needs a decision — see Open Items.

### Reputation Formula (draft)

Reputation (`R`) is a single school-level score, 0–1000, recalculated at the existing weekly payday tick (reusing that cadence rather than inventing a new one), with sports/duel results applied immediately when a match resolves so a big win feels instant rather than waiting for the weekly rollup.

```
R = Academic + Sports + Faculty − Scandal + Legacy
```

| Component | Range | Driven by |
|---|---|---|
| Academic | 0–400 | `(SchoolAverageGrade / 100) × 400` — aggregate grade average across all enrolled students. The single biggest lever, deliberately hard to move fast (System 5's design intent). |
| Sports | 0–250 | Win/loss record vs. rival schools this season, weighted toward marquee events (championships) over routine matches; a small internal-house-competition contribution is included but capped low (≤30 of the 250), since house rivalry is internal flavor, not external prestige. |
| Faculty | 0–200 | Average teacher skill level, morale, and retention — low turnover and high morale raise it; a wave of teachers quitting or burning out lowers it. |
| Scandal | −0 to −150+ (subtracted) | Discipline incidents (System 8) that go public/mishandled. A misconduct incident resolved privately and well costs little to nothing; one that escalates or leaks costs heavily. This is the main mechanical teeth behind how the player handles discipline. |
| Legacy | 0–150 | Slow-building bonus from permanent Endowment purchases and cumulative alumni success (System 11) — the main "compounds over time without resetting" lever. |

The raw score also displays as a **Rank Tier** (Unranked → Bronze → Silver → Gold → Platinum → Elite) for at-a-glance comparison against rivals, while the underlying number stays visible for granular tracking.

**What Reputation unlocks**, to keep it mechanically meaningful and not just a scoreboard number:
- Higher-quality/larger Freshman applicant pools (better predisposition stats, per System 4).
- Better teacher candidates in the hiring pool (System 1), and reduced salary demands (prestige as non-monetary comp).
- Larger/less-restricted Donor & Board of Governors funding (System 9).
- A higher sustainable tuition ceiling before enrollment suffers.
- Legacy Point generation rate scales with Rank Tier crossed — the primary ongoing source of Legacy Points, on top of discrete milestones (System 11).

### Rival Schools (draft roster)

Four named rivals (within the 3–5 target), each running a simplified version of the same formula on a periodic "rival tick" rather than a fully live simulation — nudged up/down by abstracted performance between direct matchups, with real results applied immediately when they face the player's school. Each has a distinct profile so competition has texture, matching the "everyone hates the other schools" framing:

| Rival | Personality | Strong in | Weak in |
|---|---|---|---|
| Thornwick Conservatory | Old-money, tradition-obsessed snobs | Faculty, Ritual Magic/Alchemy prestige | Sports |
| Ember Hollow Academy | Scrappy underdog, chip on its shoulder | Sports, Dueling | Academic (Faculty/Legacy) |
| Blackthorn Institute | Scandal-prone, dabbles in Dark Arts | Volatile swings — can surge or collapse from its own scandals | Consistency |
| Silverlight Preparatory | Polished PR machine, media-savvy | Presence-driven subjects (Divination/Enchanting), spins its own scandals well | Genuine substance — vulnerable if the player exposes it |

This roster is a draft for tone-setting; exact stat weightings per rival, and the specific formula for their periodic tick, should be finalized alongside the Reputation implementation phase.

## System 7: Electives — Dueling & Sports

- Dueling and Sports are **6th-period electives** — optional, only a subset of students participate.
- Only students enrolled in a sport's elective are eligible to represent the school in that sport's competitions.
- Sporting events rotate on Saturdays (System 3's weekend slot) and follow the attend-live-or-simulate pattern.
- The existing 1v1 QTE Duel modal (`duelFightModal` and related — currently a standalone Headmaster-vs-NPC combat minigame) gets folded into this: Dueling becomes a combat-magic elective/subject, and its practice sessions and matches are visitable scenes using a similar interactive-resolution style, scaled up to support team/house/school-level competitive events (not just solo Headmaster combat).
- **Dashboard "Duel Progress" widget (confirmed): removed.** The old dashboard slot (`dashDuelProgress`) that linked to the standalone duel minigame is dropped entirely rather than repurposed — sports/dueling standings and upcoming events live only within the Electives/Sports and Reputation surfaces, not on the main dashboard.
- **Existing implementation note:** house-vs-house Dueling (`runHouseDuel`/`triggerRandomHouseDuel`/`getHouseChampion`) already exists and works today (see System 6) — this system's job is to wrap that existing mechanic into the curriculum/scheduling structure (6th period, opt-in enrollment, Saturday event cadence) and extend it to school-vs-school, not to build dueling resolution logic from scratch. Skyclash (the flagship broom-sport) is still entirely novel and needs building.

## System 8: Discipline & Misconduct

- **Combined model**: a background Conduct meter (per student, possibly per staff) decays from misbehavior and drives the frequency/severity of triggered incidents; those incidents surface as queued items in the dashboard's existing Action Center / Triage Queue, each resolved via a choice-driven scene (detention, probation, private meeting, etc.), with consequences to stats/relationships.
- **Adult content is explicitly in scope**: misconduct ranges from mundane (cheating, disrespect, drama) to adult-themed (dress-code violations, streaking at sporting events, being caught in sexual acts around campus), consistent with the game's 18+ framing (all students are 18–23).
- **Dedicated intensity control**: a new setting, separate from the existing Atmosphere/Interaction Style/Consent Model sliders in the HR tab, governs how far misconduct content can go. It is explicitly *not* a hard ceiling — even a school configured as "reserved" can drift toward full corruption over time; the dial sets pace, not an absolute wall.

## System 9: Economy — Budget, Tuition, Donors

- **Tuition** (from students) is the primary revenue stream — no more stipends paid to students.
- **Teacher salaries** are the primary recurring expense, run through the reworked weekly payday cycle (System 1).
- **Bank loans are cut.** Replace Gringotts Arcane™/bonds/compounding interest with a **Board of Governors / Donors** funding mechanic — thematically a school seeking grants/endowment gifts rather than taking out predatory loans. Can still provide a lump-sum-now-for-obligations-later dynamic if useful (e.g., a donor grant with strings attached, or a fundraising drive), but framed as patronage, not banking.

## System 10: Campuses & Facilities

- Multiple physical campuses are **kept** (existing Location Subtabs / per-location funding lines in the Upgrades tab carry forward), rather than consolidating into one school with internal-only facilities. Each campus can have its own classes/staff/students, matching the existing multi-location data shape.

## System 11: Meta-Progression (replacing Prestige)

Recommended replacement for the "Graduate Now" hard-reset + Influence Points:

- **Legacy/Endowment track**: reputation milestones (ranking tiers crossed, championship wins, exceptional graduating classes) grant permanent **Legacy Points**, spent on permanent Endowment upgrades (facilities, funding baselines, starting conditions for new campuses) — all *without* wiping the current save.
- The old "Alumni" and "chat memories preserved" concepts carry forward naturally, since there's no reset to survive.
- This is a recommendation, not yet confirmed in detail with the user — flagged as an open item for a fast follow-up confirmation before implementation (see Open Items).

---

## Old → New Tab Mapping

| Old Tab | Old Purpose | New Purpose |
|---|---|---|
| Academy | Idle-clicker class buildings | Curriculum & Weekly Schedule (Systems 2–3) |
| Upgrades ("Arcane Power") | Capital upgrades + prestige reset | Faculty Development + Legacy/Endowment (Systems 1, 11) |
| Tuition | Employee-style payroll + bank loans | School Budget: tuition income, donor funding, teacher payroll (System 9) |
| People | Student roster + loose house filter | Reworked roster with clear Student/Faculty/Alumni views; richer house membership |
| HR | NPC tone/consent + admissions settings | Same purpose, lightly adjusted to match new applicant generation |
| (Duel modal) | Standalone 1v1 QTE minigame | Folded into Dueling elective + Sports (System 7) |

---

## Technical Notes & Risk

- The entire game logic currently lives in a single unminified-in-appearance-but-effectively-monolithic `<script>` block in `main.html` (one extremely long line around line 12001), authored/exported through Perchance. There is no visible modular file structure to target for "critical files" — `main.html` is the only real target, and existing function names (`switchArcanePane`, `takeLoan`, `payNow`, `toggleAutoPay`, `applyStatPreset`, `sortNewAccountantPool`, `openBonusPoolModal`, `craftCombineGifts`, etc.) are the current API surface being replaced.
- Given the size of this rework, hand-editing that single blob directly is high-risk. Before writing implementation code, a follow-up technical-planning pass should figure out how to author the new systems in a maintainable way (e.g., clearly delimited sections within the script, consistent naming/namespacing for the new data model) even if Perchance's single-file constraint remains.
- This plan intentionally stops at the design layer. Implementation should proceed in phases (suggested order: data model + Faculty hiring/progression → Curriculum/Scheduling engine → Grading formula → Budget/Donor system → Reputation/Rival schools → Electives/Sports → Discipline system → Legacy/Endowment), each phase independently testable in-browser before moving to the next, rather than attempting the full rework in one pass.
- **Scope watch on Design Pillar 1**: "every class/event is fully visitable, who's present matters everywhere" is a fairly heavy commitment toward the social/interactive end of the spectrum, even though it's scoped safely (auto-resolve by default, live-visiting is opt-in). At 50–100 students with full weekly schedules, this is the part of the plan most likely to pull implementation effort toward "social sim with a management skin" rather than the "balanced hybrid" the user asked for. Treat the end of Phase 1 (Faculty + basic Scheduling) as a checkpoint to sanity-check pacing before building out full visitability across every period.

## Open Items for Fast Follow-Up

1. **[Drafted, needs review]** Subject/class catalog (System 2) — a full 14-subject catalog with meeting patterns, year gating, and aptitude-stat tags is now drafted, along with a worked reconciliation of the weekly slot math. Needs the user's review/edits before the Scheduling engine treats it as final.
2. **[Drafted, needs review]** Reputation formula and rival-school model (System 6) — a full weighted formula, Rank Tiers, unlock effects, and a 4-school rival roster with distinct personalities are now drafted. Needs the user's review/edits, and the rival "periodic tick" formula still needs to be finalized alongside implementation.
3. **[Stub-then-refine]** Confirm the Legacy/Endowment meta-progression concept (System 11) in more detail — this was proposed, not directly asked; can launch with a placeholder and be refined later.
4. **[Stub-then-refine]** Define the Conduct meter's specific triggers/thresholds and the new dedicated intensity setting's scale/labels (System 8) — can launch with reasonable defaults and be tuned later.
5. **[Needs a decision]** Should `getHouseChampion`'s "pick any strong student" logic start respecting Dueling/Sports elective enrollment once Curriculum/Scheduling (System 3) exists, or is "any strong student can be tapped as champion" the intended feel regardless of enrollment? See System 6's "Open design tension."
6. **[Low priority, not blocking]** The word "Prefect" now means two different things in the codebase (old clicker-automation class-manager vs. new in-house student social role). Consider disambiguating one of them before either system grows further.

## Verification

Since this phase is design-only, there is no runtime verification yet. Once implementation begins, each phase should be verified by running the game locally (open `main.html` in a browser, since it's a self-contained Perchance export) and manually exercising the new tab(s)/flows before moving to the next phase.
