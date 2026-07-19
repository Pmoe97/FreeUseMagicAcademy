# FUMA Academics Overhaul — Design Plan

## Context

Free-Use Magic Academy (`main.html`, ~19.7k lines / 3.8MB single-file Perchance generator + `main.pjs` config) was ported from a prior corporate-sim project (FUOC) with a reskin but not a structural rework. The result: the game's central system — what should be "running a school" — is mechanically still "running a business":

- **Academy tab** (`academyTab`): a pure idle-clicker. "Magical classes" are Cookie-Clicker-style production buildings you buy/upgrade with a "Bulk Buy" multiplier, passively generating gold/sec across multiple "location" subtabs (old business branches).
- **Upgrades tab** ("Arcane Power"): capital-investment upgrades + a full prestige/reset loop ("Graduate Now" wipes gold/classes/progress, keeps Influence Points, Alumni, and Permanent Holdings).
- **Tuition tab**: literal old payroll — weekly "stipends" paid *to* students, auto-pay/late-penalty mechanics, an "Accountant Coverage" candidate pool, and a full bank-loan system (Gringotts Arcane™, bonds, compounding interest).

None of this reflects "headmaster manages a school." The goal of this plan is to fully replace these three tabs with a **scheduled school-management sim**: hire and develop teachers, build a real class catalog, run a weekly period-by-period schedule that students and staff actually attend, resolve grades from a genuine (gradually-influenceable) formula, build reputation against real rival schools, and handle discipline — including adult-themed misconduct — through a combined meter + interactive-incident system. This is explicitly *not* a clicker anymore; the loop is closer to a management sim with Persona-style "every period is a place you can visit" texture.

This plan is the result of extensive clarification with the user (5 rounds of questions) and is meant to be saved as a permanent design reference in the repo, followed by a phased implementation.

## Design Pillars

1. **Every period is a place.** Classes, tutoring sessions, and sporting events are all "visitable" — the Headmaster (and other characters) can attend live for an interactive scene, or let it auto-resolve via formula. Attendance is data that feeds back into outcomes.
2. **Formula-driven, but influenceable.** Grades, reputation, and match outcomes resolve from stat formulas (predisposition + skill + relationship + luck + effort), not clicks. Individual students are easy to move; moving the whole school is deliberately harder — this is the core "management" tension.
3. **Two currencies, one goal.** Money (tuition, donor funding) keeps the lights on; Reputation (grades, sports, faculty quality) is the actual growth/prestige metric, measured against real rival schools.
4. **No more hard reset.** Prestige is replaced by a persistent Legacy/Endowment system — reputation and achievements compound over time instead of wiping the save.
5. **Content intensity is a dial, not a wall.** Discipline/misconduct content has its own dedicated intensity control, independent of the general Atmosphere/Consent sliders — but even a "reserved" school can drift/corrupt over time, so the dial sets pace/ceiling, not a hard gate.

## Scope

**In scope (full rework):** Academy tab → Curriculum & Scheduling; Upgrades tab → Faculty Development + Legacy/Endowment; Tuition tab → School Budget (tuition income, donor/board funding, teacher payroll); Duel system → folded into Electives/Sports; People tab → reworked roster with clear Student/Faculty/Alumni separation; HR tab → lightly adjusted to match the new applicant-generation system (stays functionally the same purpose: shaping difficulty and admissions).

**Out of scope (untouched):** Social tab, Gifts tab, Groups tab (this is multi-person chat, not houses — confirmed via code read), Story tab, Headmaster's Tower chat. These are the relationship/roleplay layer and are not part of the "Business" gutting.

**Note on Houses:** Houses currently exist only as a loose filter attribute on the People tab roster ("All Houses" dropdown) with no dedicated management surface. This plan substantially upgrades what a House *is* (see System 8), even though no dedicated "Houses tab" is being added — house data/UI lives inside People + the new Reputation/Sports surfaces.

---

## System 1: Faculty (Teachers)

- **Hiring**: Reuse/upgrade the existing candidate-pool pattern (currently seen in Tuition's "Accountant Coverage" pool). Candidates generate deterministically and refresh instantly like today, but generation becomes thematically aware of the role/subject being hired for (not pure mad-lib). Full custom character creation remains available for both teachers and students.
- **Progression**: Full mechanical depth — teachers earn XP from teaching, level up subject mastery and a general "Teaching Skill," have a morale stat, and can retire out naturally over time. This gives payroll/attention real stakes.
- **Burnout (confirmed distinct state)**: Burnout is its own intermediate stage, not just a low-morale label. Low morale over time pushes a teacher into a "burnt out" state — reduced teaching quality, occasional erratic incidents (potentially feeding the Discipline system, System 8) — which is recoverable with attention (time off, a raise, a supportive conversation) or, if ignored, escalates to the teacher quitting.
- **Payroll**: Reuse/rework the existing weekly-payday UI (Auto-Pay toggle, Pay Early bonus, late-payment consequences) for teacher salaries only. Students no longer receive stipends — they pay tuition instead (see System 10).
- **Assignment**: A teacher is hired into a subject + skill level, then assigned to one or more class sections in the weekly schedule (System 3).

## System 2: Curriculum & Class Catalog

- Subjects span core + elective + advanced tracks. Recommend drafting a full catalog (10–16 subjects) for the user to react to in a follow-up pass — e.g. Potions, Charms, Combat Magic/Dueling, Herbology, Divination, Alchemy, History of Magic, Astronomy, Enchanting, Magical Creatures, Ritual Magic, Dark Arts (restricted/advanced), plus non-academic electives (Sports, Dueling Club).
- Each subject has difficulty tiers appropriate to student Year (Freshman intro sections vs. Graduate-level seminars), and some high-level/thematic subjects occupy **double periods**.
- Students take **6–8 subjects per semester**.

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
- **Money**: Students never receive money; they generate tuition revenue (System 10).

## System 5: Grading

- Grades resolve via a **complex weighted formula**: predisposition/aptitude (rolled trait) + accumulated skill + relationship level with the teacher/Headmaster + a luck/variance term + effort/attendance.
- Design intent: a single student is easy to move quickly (tutoring, attention, relationship investment); moving the *whole school's* average is deliberately slow and requires systemic investment (better teachers, facilities, curriculum quality) — this is the core management tension the player should feel.
- Weekend tutoring periods are a direct, visitable lever for struggling individual students.

## System 6: Reputation & Rival Schools

- Reputation is a first-class currency alongside Money, driven by aggregate grades, sports results, and faculty quality.
- **Real rival schools** (3–5), each with their own tracked reputation/stats and a visible ranking/comparison — not just flavor text.
- **Two competition layers**: Houses compete against each other *within* the school (internal rivalry, house standings), and the school as a whole competes against rival schools (external rivalry). Sporting events are the primary mechanical expression of both layers — some matches are house-vs-house, some are school-vs-school.
- **Houses get a full data model (confirmed)**: this is real added scope beyond simple visual grouping, explicitly confirmed by the user. Houses become a first-class entity with: roster membership, house points/standing (feeding internal rivalry), and a representative team pool drawn from Sports/Dueling elective enrollees (feeding external rivalry via System 7). No dedicated "Houses tab" is planned — this data surfaces inside People and the Reputation/Sports views — but the underlying model should support it properly rather than remaining a loose filter attribute.

## System 7: Electives — Dueling & Sports

- Dueling and Sports are **6th-period electives** — optional, only a subset of students participate.
- Only students enrolled in a sport's elective are eligible to represent the school in that sport's competitions.
- Sporting events rotate on Saturdays (System 3's weekend slot) and follow the attend-live-or-simulate pattern.
- The existing 1v1 QTE Duel modal (`duelFightModal` and related — currently a standalone Headmaster-vs-NPC combat minigame) gets folded into this: Dueling becomes a combat-magic elective/subject, and its practice sessions and matches are visitable scenes using a similar interactive-resolution style, scaled up to support team/house/school-level competitive events (not just solo Headmaster combat).
- **Dashboard "Duel Progress" widget (confirmed): removed.** The old dashboard slot (`dashDuelProgress`) that linked to the standalone duel minigame is dropped entirely rather than repurposed — sports/dueling standings and upcoming events live only within the Electives/Sports and Reputation surfaces, not on the main dashboard.

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

Priority order matters here: Systems 3 (Scheduling) and 5 (Grading) cannot really be implemented without #1 and #2 below, so those should be resolved first. #3 and #4 can reasonably be stubbed with placeholder values and refined later without blocking earlier phases.

1. **[Blocking]** Draft and review the full subject/class catalog (System 2) — currently only structurally defined (10–16 subjects, core/elective/advanced tiers), not the actual named list. Needed before the Scheduling engine can assign real sections.
2. **[Blocking]** Define the exact reputation formula and rival-school stat model (System 6) — needed before Grading (System 5) and Reputation can produce meaningful numbers.
3. **[Stub-then-refine]** Confirm the Legacy/Endowment meta-progression concept (System 11) in more detail — this was proposed, not directly asked; can launch with a placeholder and be refined later.
4. **[Stub-then-refine]** Define the Conduct meter's specific triggers/thresholds and the new dedicated intensity setting's scale/labels (System 8) — can launch with reasonable defaults and be tuned later.

## Verification

Since this phase is design-only, there is no runtime verification yet. Once implementation begins, each phase should be verified by running the game locally (open `main.html` in a browser, since it's a self-contained Perchance export) and manually exercising the new tab(s)/flows before moving to the next phase.