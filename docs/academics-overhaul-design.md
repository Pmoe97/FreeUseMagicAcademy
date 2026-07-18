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

**Existing implementation (first pass, shipped):** `processFacultyPayroll()` runs on the existing Friday tick inside `onDayChange()`, alongside (but not gated by) the old student-stipend `runFridayTuition()`/Auto-Pay flow — teacher salaries are paid unconditionally each Friday rather than waiting on that toggle. It sums each faculty member's `salary` (annual figure) / 52 across `gameState.faculty`, deducts from `gameState.gold`, and mirrors `processWeeklyTuition()`'s existing shortfall pattern: on insufficient gold it drains to 0 and books the remainder to `gameState.facultyPayroll.arrears`, plus applies a flat −15 morale hit to every faculty member that week (floored at 0) — a minimal tie-in to the Burnout design above, not a full burnout state machine. History is tracked in `gameState.facultyPayroll.history` (capped at 12 weeks) for future UI. The Faculty band now shows a one-line weekly-payroll-cost summary (and arrears, if any) alongside the existing curriculum-summary line. **Not yet built:** a dedicated payroll UI (the old Tuition-tab-style Auto-Pay/Pay-Early/history surface) — this first pass only wires the money and the morale consequence, reusing the *cadence*, not the *UI*, of the old system; a real Faculty-specific payday UI is part of the eventual full System 9 Tuition-tab replacement.
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

**Existing implementation (first pass, shipped):** the engine now exists — `gameState.sections`, `createSection()` (auto-opens on hire, picks day-pattern from `SUBJECTS`/period via a least-congested heuristic), `autoEnrollStudentInSections()` (year-gated, conflict-aware, double-periods and scarce-section subjects prioritized), `student.year`/`rollStudentYear()`, and `getStudentWeeklyGrid()`. This is a **greedy heuristic, not an optimal solver** — it won't always find a perfect-fit schedule when one exists, though testing shows it handles realistic cases (including trading off a core class to make room for a double period) reasonably well. Manual override and the actual Academy-tab replacement are still open.

**Weekly-grid UI (first pass, shipped):** two read-only modal views, both built directly on `gameState.sections`/`enrolledStudentIds` with no denormalized copy, so they cannot drift apart — `showTeacherScheduleModal(teacherId)` (a Mon–Fri × Period macro grid via the new `getTeacherWeeklyGrid()`, reachable from a "📅 Schedule" button on each Faculty band row) and `showStudentScheduleModal(studentId)` (the same grid renderer over the existing `getStudentWeeklyGrid()`, reachable via a new "🔍 Look Up a Student's Schedule" search modal, also in the Faculty band). Clicking any filled period on *either* grid opens `showSectionRosterModal()`, showing the section's full enrolled roster — the same modal from both directions, which is what guarantees the teacher-side and student-side views agree. This still lives entirely off the Faculty band; it is not yet integrated into the People tab or a real Academy-tab replacement, and weekend tutoring/sporting periods aren't modeled (System 3's 5th/weekend periods aren't part of `gameState.sections` yet).

**Live-refresh readiness (shipped, groundwork for manual editing):** `scheduleModalContext` tracks which schedule view is currently open (teacher grid / student grid / roster / lookup / add-class / class-options) and `refreshOpenScheduleModal()` re-renders it in place from live `gameState`; wired into both hire/fire faculty mutation points. Also fixed as part of this: `fireFaculty()` previously left the teacher's sections orphaned in `gameState.sections` (a pre-existing gap, not introduced by the schedule UI) — it now removes them and cleans up affected students' `scheduledSectionIds` too, since a dangling section would otherwise silently break the "faithful schedule" guarantee the moment a teacher was dismissed.

**Manual schedule editing (first pass, shipped):** `showStudentScheduleModal()` is now always editable — clicking a filled period opens `showClassOptionsModal()` (View Full Roster / Drop This Class), clicking an open period opens `showAddClassModal()`, which lists candidate sections via `sectionCoversSlot()` filtered to ones the student isn't already in, isn't full, and doesn't conflict with (reusing the existing `studentHasConflict()`/`enrollStudentInSection()` engine functions directly, so a manually-added class is held to the exact same physical constraints — no double-booking, no over-capacity — as an auto-enrolled one). Deliberately *not* re-enforced here: the auto-enroll heuristic's subject/year-targeting preferences — those are soft defaults for automatic placement, not hard rules, so the headmaster can manually place a student in any open, non-conflicting section regardless of year, matching the design doc's "manual override" intent (e.g. Ritual Magic's sign-off gate). All mutation paths (`addStudentToSectionUI`, `dropStudentFromSectionUI`) navigate back into a freshly-rendered `showStudentScheduleModal()`, and `refreshOpenScheduleModal()` covers the two new transient modal types too. **Entry points:** the Faculty band's teacher rows/lookup button (as before), plus a new "📅 Schedule" action in the People tab's existing per-student overflow menu (`data-action="schedule"`, wired through the existing `handleStudentAction()` dispatcher rather than a hand-rolled onclick, to avoid touching `updatePeopleTab()`'s large per-card template directly).
- **Attendance & visitability**: Every scheduled period — a class section, a tutoring session, a sporting event — is a "place." Default resolution is automatic/formula-driven (for scale: 50–100 students × 5 periods × 5 weekdays is far too much to force manual play on every instance). The player (and potentially other characters) can choose to "walk into" any specific instance for a live interactive scene; who is actually in attendance for that instance modifies the formula outcome for everyone present. Saturday sporting events follow the same pattern explicitly: attend live, or skip and let it simulate.

## System 4: Students & Lifecycle

- **Scale**: Target 50–100 concurrently enrolled students at launch, with the data model built to scale further (some students "featured"/fully detailed, especially those with active relationships; others lighter-weight/background, promotable to featured on demand).
- **Full lifecycle**: Students age up through Freshman → Sophomore → Junior → Senior → Graduate over defined in-game time. After Graduate year they leave as Alumni (existing concept, preserved from the old prestige modal) or become hireable Faculty (per System 1). New Freshman cohorts enroll each cycle to keep the population steady.
- **Money**: Students never receive money; they generate tuition revenue (System 9).

## System 5: Grading

- Grades resolve via a **complex weighted formula**: predisposition/aptitude (rolled trait) + accumulated skill + relationship level with the teacher/Headmaster + a luck/variance term + effort/attendance.
- Design intent: a single student is easy to move quickly (tutoring, attention, relationship investment); moving the *whole school's* average is deliberately slow and requires systemic investment (better teachers, facilities, curriculum quality) — this is the core management tension the player should feel.
- Weekend tutoring periods are a direct, visitable lever for struggling individual students.

**Existing implementation (first pass, shipped):** the formula is implemented as `calculateSectionGrade(student, section)`, run per enrolled section, weighted 30% aptitude / 25% teacher quality / 15% relationship / 15% effort / 15% luck, clamped to an integer 0–100. Rather than inventing parallel data structures, each term reuses what already exists elsewhere in the codebase:
- **Aptitude** — the one genuinely new piece of state: `student.aptitudes`, a one-time-rolled (20–80) value for each of the 6 stats `SUBJECTS` already tags classes with (Intellect/Focus/Creativity/Willpower/Presence/Physicality — System 2's proposed stat list), lazily rolled via `ensureStudentAptitudes()`. `getStudentAptitudeForSubject()` averages whichever 1–2 stats the subject is tagged with.
- **Accumulated skill** — read from the *teacher's* side (`teachingSkill`/`subjectMastery`, System 1's Faculty progression fields) via `getTeacherQualityScore()`, rather than adding a second, parallel per-student skill-growth system. This also makes the `strict` faculty trait's description ("Higher average grades") mechanically true for the first time — it had existed as inert flavor text since the Faculty system shipped, with no grading system yet to make it real; the other trait descriptions (`inspiring`, `researcher`, `studentFavorite`, `byTheBook`) are still inert, since they depend on XP-gain/relationship-growth/burnout systems that aren't built yet.
- **Relationship "with the teacher/Headmaster"** — the doc's wording explicitly allows either; this reuses the existing `calculateAverageRelationship(student)` (the student's general Headmaster bond), since there's no per-student-per-teacher relationship tracking and building one was out of scope for this pass.
- **Effort/attendance** — reuses `stats.diligence`, the existing FUOC-era stat already shown in the People tab as an effort indicator. There's no actual attendance data yet (visitability/attendance isn't built — see Technical Notes), so this is a stand-in until that exists.
- **Luck/variance** — a fresh random roll each time the formula runs.

Grades are **persisted, not computed live** — `processWeeklyGrading()` runs on the existing Friday tick (same cadence System 6's Reputation formula already says it wants to reuse) and writes `student.grades[subjectId]`, so a student's grade is a stable number the player can check between paydays rather than jittering every render. `getStudentGPA()` averages a student's own grades; `getSchoolAverageGrade()` averages all active students' GPAs — this is exactly the `SchoolAverageGrade` term System 6's draft Reputation formula already names, so Reputation's Academic component can call it directly once that system is built. `getLetterGrade()` maps to a standard A/A-/B+/.../F scale (deliberately not Hogwarts' O.W.L. lettering, matching the game's existing practice of avoiding direct HP-canon terminology, e.g. the renamed Houses).

**UI (shipped):** grade badges appear directly on a student's own schedule grid cells (the System 3 UI already built), and a dedicated `showStudentGradesModal(studentId)` report (overall GPA + per-subject letter/percent) is reachable from a new "🎓 Grades" action in the People tab's overflow menu, following the same `data-action`-dispatch pattern as the "📅 Schedule" entry added alongside it. Wired into `scheduleModalContext`/`refreshOpenScheduleModal()` like every other schedule-adjacent view.

**Not yet built:** real attendance (depends on the Attendance & Visitability system above, which is unbuilt), weekend tutoring as a grade lever (depends on weekend periods existing in `gameState.sections`, which they don't yet), and any UI trend/history (only the current week's grade is kept, no history array).

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

**Still to build (this plan's actual remaining scope for System 6):** ~~the *external* rival-school layer (3–5 named rival schools, their own reputation tracking, school-vs-school competition) — nothing like `RIVAL_SCHOOLS` exists yet, confirmed via code search.~~ **Shipped, first pass — see the "Existing implementation" note under the Reputation Formula below.** The Reputation formula treats `gameState.houseStandings` as the data source for its internal-house-competition sub-component rather than inventing a parallel points system, as originally planned here.

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

**Existing implementation (first pass, shipped):** `calculateSchoolReputation()` implements this formula, but only 3 of the 5 components can be genuinely computed today — the other two are honestly stubbed at 0 rather than faked, since their source systems don't exist yet:
- **Academic** (0–400, fully live): `getAcademicReputationComponent()` = `(getSchoolAverageGrade() / 100) × 400`, directly consuming System 5's Grading output — exactly the `SchoolAverageGrade` term this draft already named.
- **Faculty** (0–200, fully live): `getFacultyReputationComponent()` blends average `getTeacherQualityScore()` (System 5's teacher-quality helper, 70% weight) with average `morale` (30% weight) across the roster. "Retention" (the third named driver) isn't tracked as a rate anywhere yet, so it's not part of the blend — a documented gap, not a silent omission.
- **Sports** (0–30 of the stated 0–250, partially live): `getSportsReputationComponent()` only counts the internal house-cup sub-component the doc already says to cap "≤30 of the 250" — it sums `gameState.houseStandings` (already built, driven by `runHouseDuel`) and scales/clamps to that cap. The other ~220 points, from *external* school-vs-school results, can't exist until Electives/Sports (System 7) does — Skyclash isn't built and house dueling isn't gated by elective enrollment (see the still-open "Open design tension" above).
- **Scandal** (stubbed at 0): needs System 8 (Discipline & Misconduct), not built.
- **Legacy** (stubbed at 0): needs System 11 (Meta-Progression), not built.

Recalculated weekly via `processWeeklyReputation()` on the same Friday tick as Payroll/Tuition/Grading (matching this draft's own "reuse that cadence" note), after Grading runs so Academic reflects that week's fresh grades, and after Payroll so a morale hit from a missed paycheck shows up in Faculty immediately. `getReputationRankTier()` implements the tier bands above with placeholder-but-documented thresholds (100/300/500/700/900) — not confirmed with the user, easy to retune. None of the "What Reputation unlocks" bullets below are wired up yet (applicant pools, hiring-pool quality, donor funding size, tuition ceiling, Legacy generation) — the score exists and is visible, but nothing else in the game reacts to it yet.

**Rival Schools — existing implementation (first pass, shipped):** `RIVAL_SCHOOLS` is the 4-school roster from the draft table below, each with a `baseReputation` starting value and a `volatility` (Blackthorn's is highest, matching its "can surge or collapse" flavor). `processRivalSchoolsTick()` runs the same Friday cadence, nudging each rival's `gameState.rivalSchools[id].reputation` by a random walk of ±volatility, clamped to [0,1000] — this is the "abstracted performance between direct matchups" the draft describes; there are no actual matchups yet since those need Sports (System 7). `getReputationLeaderboard()` ranks the player's school against all 4 rivals together. **UI:** a `showReputationModal()` comparison view (score + component breakdown + ranked leaderboard with each rival's personality/strong/weak blurb), reachable from a new "🏆 Reputation: N · Tier" button in the Faculty band, following the same pattern as the Payroll/Tuition/Donor lines already there.

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

This roster was tone-setting; it's now the literal shipped `RIVAL_SCHOOLS` catalog (see the "Existing implementation" note above) — exact numeric weightings (base reputation, volatility) are first-pass placeholders, not confirmed with the user, easy to retune once real school-vs-school matches (System 7) give something to calibrate against.

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

**Existing implementation (first pass, shipped):** `processWeeklyTuitionIncome()` runs on the same Friday tick as Payroll/Grading, adding `getStudentTuitionRate(student)` (a modest per-year-scaled rate, 20–40/week, same small-dollar economy scale as Faculty salaries — not the old stipend system's much larger numbers) for every active student straight to `gameState.gold`. `requestDonorGrant()` is a player-triggered, cooldown-gated (14 days) lump sum sized off current school scale (`calculateDonorGrantAmount()`: faculty count + student count) — deliberately not Reputation-driven yet since System 6 doesn't exist; swap that formula once it does. Both surface in the Faculty band alongside the existing Payroll line.

**How the old stipend/loan system was retired:** rather than hand-editing the old code (high-risk in this file — see Technical Notes), every live entry point was disconnected instead: `onDayChange`'s Friday block no longer calls `runFridayTuition`/`showTuitionModal`/`processLoanInterest`/`processTuitionConsequences`/`generateStipendIncreaseRequests`/`generateAdvanceRequests`; `takeLoan()` and `openPelicanModal()` (all 4 loan types' only entry points) now return an in-fiction redirect notification before doing anything else; and `processWeeklyTuition()` itself (the old stipend-payout function) got the same early-return guard as a belt-and-suspenders measure, since it turned out to still be reachable from the old Tuition tab's "Pay Now"/"Pay Early" buttons independent of the Friday tick. All the old functions and their UI (the `tuitionTab`, its Accountant Coverage pool, Auto-Pay toggle, credit rating, loan buttons) are still physically present and navigable — just inert. **Not yet built:** the real "School Budget" tab UI that's supposed to replace it (same open gap as the Academy tab's Curriculum/Scheduling replacement) — this pass only fixed the money flow and disconnected the old mechanics, it didn't build or hide any tab UI.

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
