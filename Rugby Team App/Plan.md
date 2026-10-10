# Rugby Team App — Plan

> Status: **planning only**. No code yet. Base project: [openGym](https://github.com/DuarteSantos8/openGym) (self-hosted gym tracker, AGPL-3.0).
> Visual map: [[UX & Feature Map.canvas]]

## 1. What we're building

One app for a rugby squad that covers four things:

| Pillar | What it does | Where it comes from |
|---|---|---|
| **Chat** | Team, unit (forwards / backs) and DM channels, announcements, threads that hang off an event | New |
| **Calendar** | Fixtures, trainings, gym slots, socials, availability; one team calendar plus a personal view | New, with openGym's weekday plan as a starting point |
| **Practice scheduling** | Coaches build sessions (drills, time blocks, units), publish them, players RSVP, coaches take attendance | New |
| **Workout tracking (open gym)** | S&C programmes assigned per position group, players log in the gym, coaches see who did what | **openGym**: guided workouts, progression, 1RM, PRs, rest timer, stats |

## 2. Starting from openGym: what we keep and what we add

**Keep as-is (training engine):**
- Exercise library (5,600+), routines, rotation plans, supersets, timed and cardio sets
- Guided workout screen, rest timer, plate math, RIR/RPE effort
- Progression engine (`lib/progression.js`), 1RM (`lib/onerm.js`), recovery model, stats/heatmap
- Passkey login, PWA + Capacitor (Android/iOS), Web Push notifications
- Optional AI coach + read-only MCP bridge

**Gaps for a team app:**
- openGym is **single-user**. Each user has their own `state-<uid>.json`, and there's no concept of a team, role or shared data.
- Storage is **flat JSON files**. That's fine for one lifter. It won't hold up under chat volume or many concurrent team writes.
- There's **no real-time channel**. Sync is debounced push/pull of the whole state.
- The upstream roadmap edges closer: *groups* (v1.4.4), *sharing & gyms* (v1.4.5), the *trainer role* where one person writes another's plan (v1.4.7), and *database storage* (later). It's worth tracking or contributing to rather than diverging.

**Decision to make (recommendation: fork + team layer):**
1. **Fork openGym and add a team layer.** Reuse the training engine and UI. Add `teams`, roles, events, chat and a real database. *Recommended*: the workout piece is the hardest part and it's already done.
2. **Separate team app plus openGym as a linked service.** A cleaner split, but two logins and two apps for players.
3. **Greenfield.** Only worth it if we drop self-hosting.

> AGPL-3.0: if we run a modified openGym as a network service, we must publish our source. Fine for a club app, but decide it up front.

## 3. Users and roles

| Role | Main jobs |
|---|---|
| **Head coach** | Plans the season, builds practices, selects the team, posts announcements |
| **S&C coach** | Writes gym programmes per position group, watches compliance and load |
| **Captain / leadership** | Posts in chat, nudges attendance, sees the squad view |
| **Player** | Sees schedule, RSVPs, chats, logs open-gym workouts, reports wellness |
| **Physio / medic** | Sets injury status (available / modified / out), return-to-play notes |
| **Team manager / admin** | Roster, invites, fixtures, venues, permissions |

Rugby-specific attributes on a player: **position** (1–15 + utility), **unit** (forwards / backs; front row, back row, halves, etc.), **squad** (1st XV, 2nd XV, U18…) and **availability status**.

## 4. Feature map

```mermaid
mindmap
  root((Rugby Team App))
    Chat
      Team channel
      Unit channels — forwards / backs / front row
      Direct messages
      Announcements — coach only, pinned, read receipts
      Event threads — auto-created per practice / match
      Polls — e.g. kit colour, social date
      Media — clips, lineout calls
    Calendar
      Team calendar — month / week / agenda
      Event types — training, match, gym, meeting, social
      Availability — Yes / No / Maybe + reason
      Personal view — my events + my gym plan
      Reminders — push before kick-off / session
      Export — iCal / Google subscription
    Practice Scheduling
      Session builder — time blocks, drills, units
      Drill library — set piece, defence, attack, conditioning
      Templates — Tuesday contact, Thursday captain's run
      Publish → calendar + chat thread
      RSVP and attendance
      Post-session RPE → training load
    Open Gym — openGym core
      Programmes per position group
      Guided workout + logging
      Progression, 1RM, PRs
      Gym check-in / open-gym slots
      Coach compliance dashboard
      Fitness testing — Bronco, Yo-Yo, 1RM, sprint
    Squad & Team
      Roster and positions
      Roles and permissions
      Injury / availability status
      Team selection — match-day 23
      Invites — QR / code
    Insights
      Attendance %
      Load — sRPE × minutes, acute:chronic
      Gym compliance
      Leaderboards — opt-in
```

## 5. Information architecture (navigation)

Bottom tab bar, five tabs. That's the same pattern openGym uses (`components/TabBar.jsx`).

```mermaid
flowchart TD
    Login[Login / passkey] --> Join{Has team?}
    Join -- no --> JoinTeam[Join team by code / QR] --> Profile[Set position, unit, squad]
    Join -- yes --> Home
    Profile --> Home

    subgraph Tabs
      Home[🏠 Home<br/>today: next event, RSVP due, today's gym session, unread]
      Calendar[📅 Calendar<br/>month / week / agenda]
      Chat[💬 Chat<br/>channels, DMs, threads]
      Gym[🏋️ Gym<br/>my programme, start workout, history, stats]
      Team[👥 Team<br/>roster, selection, coach tools]
    end

    Home --> EventDetail
    Calendar --> EventDetail[Event detail<br/>plan, location, RSVP, attendees, thread]
    EventDetail --> EventThread[Event chat thread]
    Chat --> EventThread
    Gym --> Workout[Guided workout — openGym]
    Workout --> Finish[Finish → session RPE → PRs]
    Gym --> Stats[Stats / PRs / tests]
    Team --> Roster[Roster & player profile]
    Team --> CoachTools[Coach tools]
    CoachTools --> PracticeBuilder[Practice builder]
    CoachTools --> ProgrammeBuilder[Gym programme builder<br/>openGym RoutineEdit]
    CoachTools --> Selection[Team selection]
    CoachTools --> Dashboard[Squad dashboard<br/>attendance, load, compliance]
    PracticeBuilder --> Publish[Publish] --> Calendar
    Publish --> Chat
```

## 6. Key user flows

### 6a. Coach schedules a practice → player turns up

```mermaid
sequenceDiagram
    actor Coach
    participant App
    actor Player
    Coach->>App: Practice builder: pick template, set date/time/venue, add drill blocks
    Coach->>App: Publish
    App->>App: Create calendar event + event chat thread
    App-->>Player: Push "Training Tue 18:30 — RSVP"
    Player->>App: RSVP Yes / No (+ reason)
    App-->>Coach: Live availability count by unit
    App-->>Player: Reminder 2h before
    Coach->>App: Take attendance at the session (tick list / QR check-in)
    App-->>Player: Prompt "How hard was it? (RPE 1–10)"
    Player->>App: RPE 7
    App->>App: Load = RPE × minutes → player load history
```

### 6b. Player does open gym

```mermaid
sequenceDiagram
    actor SC as S&C Coach
    participant App
    actor Player
    SC->>App: Assign "Forwards Strength — Block 2" to Forwards group
    App-->>Player: Programme appears in Gym tab
    Player->>App: Check in to open-gym slot (optional)
    Player->>App: Start today's workout (openGym guided flow)
    App->>Player: Pre-filled weights, rest timer, PR detection
    Player->>App: Finish + session RPE
    App->>App: Progression engine advances weights
    App-->>SC: Compliance dashboard updates (done / missed / partial)
    App-->>Player: Optional post to team channel "New squat PR 🎉"
```

### 6c. Match week

```mermaid
flowchart LR
    A[Fixture in calendar] --> B[Availability poll Mon]
    B --> C[Physio marks injuries]
    C --> D[Coach picks 23 — Team selection]
    D --> E[Team sheet posted to Announcements]
    E --> F[Captain's run Fri]
    F --> G[Match day: meet time, kit, venue]
    G --> H[Post-match RPE + minutes played]
    H --> I[Load feeds next week's gym plan]
```

## 7. Feature → screen matrix

| Feature | Home | Calendar | Chat | Gym | Team | Role |
|---|:-:|:-:|:-:|:-:|:-:|---|
| Next event card + RSVP | ● | ● | | | | all |
| Event chat thread | | ● | ● | | | all |
| Announcements | ● | | ● | | | coach post / all read |
| Practice builder | | ● | | | ● | coach |
| Attendance | | ● | | | ● | coach |
| Gym programme | ● | ○ | | ● | ● | S&C writes / player runs |
| Guided workout (openGym) | ● | | | ● | | player |
| Compliance + load dashboard | | | | | ● | coaches |
| Injury status | | ○ | | | ● | physio |
| Team selection | | ● | ● | | ● | coach |
| Fitness tests | | ● | | ● | ● | S&C |

● primary · ○ secondary

## 8. Data model (draft)

```mermaid
erDiagram
    TEAM ||--o{ MEMBERSHIP : has
    USER ||--o{ MEMBERSHIP : joins
    MEMBERSHIP {
      string role "head_coach|sc_coach|captain|player|physio|manager"
      string position "1-15 / utility"
      string unit "forwards|backs"
      string squad "1st XV, 2nd XV..."
      string status "available|modified|injured"
    }
    TEAM ||--o{ GROUP : defines
    GROUP }o--o{ MEMBERSHIP : contains
    TEAM ||--o{ EVENT : schedules
    EVENT {
      string type "training|match|gym|meeting|social"
      datetime start
      datetime end
      string venue
    }
    EVENT ||--o| PRACTICE_PLAN : has
    PRACTICE_PLAN ||--o{ DRILL_BLOCK : contains
    DRILL_BLOCK }o--|| DRILL : uses
    EVENT ||--o{ RSVP : collects
    EVENT ||--o{ ATTENDANCE : records
    EVENT ||--o| CHANNEL : thread
    TEAM ||--o{ CHANNEL : has
    CHANNEL ||--o{ MESSAGE : contains
    USER ||--o{ MESSAGE : writes
    TEAM ||--o{ PROGRAMME : publishes
    PROGRAMME }o--o{ GROUP : assigned_to
    PROGRAMME ||--o{ ROUTINE : "openGym routines"
    USER ||--o{ WORKOUT_LOG : "openGym session"
    WORKOUT_LOG }o--o| ROUTINE : follows
    USER ||--o{ LOAD_ENTRY : "sRPE x min"
    EVENT ||--o{ LOAD_ENTRY : generates
    WORKOUT_LOG ||--o| LOAD_ENTRY : generates
```

## 9. Architecture sketch

```mermaid
flowchart LR
    subgraph Client["Client — React PWA + Capacitor (from openGym)"]
      UI[Views: Home, Calendar, Chat, Gym, Team]
      Store[Zustand store<br/>personal workout state]
      RT[Realtime client<br/>SSE / WebSocket]
    end
    subgraph Server["API — fork of openGym api/"]
      Auth[Passkeys + password/OIDC]
      Train[Workout sync — existing]
      TeamAPI[Teams, roles, events, RSVP]
      ChatAPI[Chat + fan-out]
      Push[Web Push / FCM / APNs]
    end
    DB[(Postgres or SQLite<br/>teams, events, chat)]
    Files[(DATA_DIR JSON<br/>per-user workout state)]
    UI --> Store --> Train --> Files
    UI --> TeamAPI --> DB
    RT <--> ChatAPI --> DB
    TeamAPI --> Push
    ChatAPI --> Push
```

Notes:
- **Leave workout state where it is at first** (per-user JSON plus openGym's sync). Put the new team data in a real DB. Move workout logs into the DB later if the coach dashboards need cross-player queries. Upstream's "database storage" milestone may get there first.
- **Real-time chat**: SSE plus POST works with zero new deps on `node:http`. A WebSocket lib is the alternative. openGym is strict about dependencies, which matters if we want to merge upstream.
- **Push**: reuse openGym's Web Push. Native shells need FCM/APNs for reliability.

## 10. Phased roadmap

| Phase | Scope | Exit criteria |
|---|---|---|
| **0 — Foundations** | Fork, team + membership + roles, invite by code/QR, roster, DB added | Coach creates team, 30 players join |
| **1 — Calendar & practice (MVP)** | Events, RSVP, reminders, practice builder + templates, attendance | One full training week runs through the app |
| **2 — Chat** | Team, unit and DM channels, announcements, event threads, push | WhatsApp group not needed for logistics |
| **3 — Open gym for teams** | Programmes per group (built on openGym), compliance dashboard, session RPE, check-in | S&C sees weekly compliance per unit |
| **4 — Match week & insights** | Selection/team sheet, injury status, load (sRPE, ACWR), fitness tests, iCal export | Coach plans match week end-to-end |
| **Later** | Video clips, polls, leaderboards, GPS/wearable import, multi-squad clubs, payments/subs | — |

## 11. Open questions

1. Self-hosted per club (openGym's model) or one hosted service for many teams?
2. Fork and diverge, or contribute the team layer upstream (groups, trainer role, sharing are already on their roadmap)?
3. Does chat need to be E2E encrypted, or is server-side storage OK? (Minors/U18 squads raise safeguarding rules: no adult↔minor DMs, message retention.)
4. Which DB: SQLite (simple self-host) or Postgres (hosted, multi-team)?
5. Will players accept coaches seeing their gym logs? We need clear privacy controls. Default: coaches see team-programme sessions only.
6. Integrations: Google Calendar sync, wearables (GPS/heart rate), club management systems?

## 12. Going commercial: workout tracking as one module

Visual version: https://claude.ai/artifact/6sVCMYiqccQW4cLhW2sfdT

**Reframe:** the product is a rugby team platform. Workout tracking is one module beside chat, schedule, practice planner, selection, welfare/load, club admin and payments. Every module runs on a shared core: identity, teams/roles, notifications, data, integrations and billing.

| Plan | Includes |
|---|---|
| Free | Chat, schedule, RSVP, roster, reminders |
| Pro (per team) | Practice planner, selection, workout tracking, welfare & load, coach dashboard |
| Club (per club) | Pro plus many squads, safeguarding roles, registration, payments, reports |

Revenue: subscriptions (coaches/clubs pay, players don't), a fee on in-app payments, a programme marketplace for S&C coaches, and later licences for unions and leagues.

**Licensing (summary of openGym's LICENSE/NOTICE.md, not legal advice):**
- Code is AGPL-3.0. Selling a hosted modified version means publishing all of its source, including anything linked into it.
- The exercise animations are Gym visual's, not AGPL. NOTICE.md says a commercial fork needs its own licence from gymvisual.com.
- **Recommendation:** build our own workout module and use openGym only as a UX reference. Alternatives: a dual licence from the author (only covers code he owns), or ship everything as AGPL open-core and sell hosting.
