# Learning System Architecture

**Status:** Proposed foundation  
**Branch:** `feature/learning-foundation`  
**Product principle:** A space for thought.

This document defines the boundaries, concepts, relationships, and initial implementation scope for Odelis Learning. It is an architectural guide, not a commitment to implement every capability at once.

## 1. Purpose

Learning is the structured environment for understanding, practicing, reviewing, and tracking knowledge over time.

It should support independent study across different subjects while connecting naturally to the rest of Odelis. A learner should be able to organize a subject, study a topic, create learning materials, practice, and understand their progress without needing another module to function.

## 2. Module boundaries

Learning owns the following concepts:

- **Subjects and topics:** the structure of what a person is learning.
- **Flashcards:** prompts and answers organized into decks.
- **Quizzes and attempts:** practice activities and their recorded results.
- **Reviews:** scheduled and completed spaced-repetition reviews.
- **Study sessions:** records of focused learning activity.
- **Learning progress:** activity, performance, retention estimates, and mastery indicators.
- **Learning paths:** ordered topics and milestones toward a learning goal.

Other modules remain responsible for their own concepts:

- **Notes & Library** owns notes, collections, and saved resources.
- **Planner** owns tasks, deadlines, calendar entries, and time blocks.
- **Projects** owns projects, milestones, and project organization.
- **Focus** owns focus timers and focus-session mechanics.
- **Overview** presents summaries from other modules; it should not become a second source of truth.

Modules may reference one another through stable IDs. They should not create duplicate copies of the same underlying record.

## 3. Core learning concepts

### Subjects and topics

A subject is a broad area of study, such as Mathematics or History. A topic is a specific area within a subject.

Topics may be nested to allow useful levels of detail. For example:

```text
Mathematics
└── Algebra
    └── Quadratics
        └── Factoring
```

Each subject and topic should have a stable ID, a name, optional description, creation and update timestamps, and an optional ordering value.

A topic may be associated with notes, resources, decks, quizzes, and study sessions. These associations should use IDs rather than embedding duplicate records.

### Flashcards and decks

A deck groups related flashcards. Each card contains a prompt and an answer, and may optionally include a hint, explanation, tags, or a topic association.

The first version should support simple text-based cards. More complex media, cloze deletion, and card templates can be considered later.

A card's content and its review history are separate concerns: editing a card should not erase its historical review records.

### Quizzes and attempts

A quiz is a reusable set of questions. An attempt is one learner's completed or in-progress response to that quiz.

The initial question types are:

- Single-answer multiple choice
- Multiple-select
- True or false
- Short written response

An attempt should preserve the question version or relevant question snapshot used at the time. This prevents later quiz edits from silently changing the meaning of historical results.

### Reviews and spaced repetition

A review records a learner's interaction with a flashcard. A review may include when it occurred, the learner's difficulty rating, and the next scheduled review date.

Scheduling logic should be isolated from the interface and designed so that the algorithm can evolve without rewriting card data or review history.

The initial implementation should favor a transparent, understandable scheduling approach over a complicated adaptive algorithm. The user should be able to see what is due and understand why a card is scheduled.

### Study sessions

A study session records a period of learning activity. It may be associated with a subject, topic, deck, quiz, task, or project.

A session can record its start and end times, duration, activity type, and optional notes. It should distinguish time spent from evidence of learning: time alone must not be treated as mastery.

### Progress and analytics

Learning analytics should distinguish several different signals:

- **Activity:** what the learner has worked on and how often.
- **Performance:** results on quizzes and practice.
- **Review history:** how cards have been rated and scheduled.
- **Retention estimate:** an estimate derived from review history, clearly labeled as an estimate.
- **Mastery:** a cautious topic-level indicator based on multiple forms of evidence, not simply time spent or a single quiz score.

The interface should explain what each measure represents. Avoid presenting a precise mastery percentage when the underlying evidence does not justify that precision.

### Learning paths

A learning path is an optional sequence of topics and milestones toward a learning goal. Topics remain valid independently of any path and may appear in more than one path.

Learning paths are not required for the first usable version of Learning.

## 4. Relationships between records

The following relationships are the intended conceptual model:

- Subject → Topics
- Topic → Notes and resources (by reference), decks, quizzes, and study sessions
- Deck → Flashcards
- Flashcard → Review history
- Quiz → Questions
- Quiz → Quiz attempts
- Study session → optional subject, topic, task, or project references
- Learning path → ordered topic and milestone references
- Planner task → optional subject or topic reference
- Focus session → optional task reference

Relationships are optional where appropriate. A person should be able to create a flashcard or study session without first building a complete subject hierarchy.

## 5. Local-first data architecture

The first implementation is local-first:

1. The app works without an account or network connection.
2. User data is stored on the user's device.
3. Records use stable IDs so that relationships remain intact when data is edited or moved.
4. Data changes are saved through a shared storage service rather than scattered direct storage calls.
5. The app provides a reliable export and import path before the data model grows substantially.
6. Future cloud synchronization is a later capability, not a prerequisite for Learning.

The current application uses browser local storage in multiple areas. The implementation should introduce a storage boundary and migrate existing data carefully. It must not casually rename or discard existing task, schedule, or note storage keys.

### Data safety requirements

- Validate imported data before replacing existing records.
- Do not overwrite current data until an import has passed validation and the user confirms the action.
- Keep export data portable and documented.
- Include a schema version in backup data.
- Plan explicit migrations when the schema changes.
- Handle missing or malformed stored data gracefully; do not silently reset a user's workspace.
- Keep review and attempt history when editable content changes.

For the first stage, browser local storage may remain the persistence mechanism behind the storage service. If the volume or complexity of data later requires it, the storage implementation can move to IndexedDB without changing every module's interface.

## 6. Suggested conceptual record shape

These examples describe the intended fields, not finalized code or a required serialization format.

| Record | Core fields |
|---|---|
| Subject | id, name, description, createdAt, updatedAt |
| Topic | id, subjectId, parentTopicId, name, description, order |
| Deck | id, title, topicIds, createdAt, updatedAt |
| Flashcard | id, deckId, prompt, answer, explanation, tags |
| Review | id, flashcardId, reviewedAt, rating, nextReviewAt |
| Quiz | id, title, topicIds, questionIds, updatedAt |
| Quiz question | id, type, prompt, choices, expectedAnswer, explanation |
| Quiz attempt | id, quizId, startedAt, completedAt, responses, result |
| Study session | id, startedAt, endedAt, activityType, relatedIds |
| Learning path | id, title, goal, orderedTopicIds, milestones |

Use ISO 8601 timestamps and stable string IDs. Exact field names and validation rules should be finalized alongside the implementation.

## 7. Initial implementation scope

Build the foundation in this order:

1. **Shared data and storage foundation**
   - Define ID and timestamp conventions.
   - Establish a Learning data schema and validation.
   - Create a storage service boundary.
   - Add versioned backup/export and safe import behavior.
   - Preserve existing Planner and Notes & Library data.

2. **Subjects and topics**
   - Create, edit, organize, and remove subjects and topics.
   - Support nested topics.
   - Provide useful empty states.

3. **Topic-linked learning materials**
   - Connect existing Notes & Library records to topics by ID.
   - Add flashcard decks and simple text-based cards.

4. **Practice and review**
   - Add basic quizzes and recorded attempts.
   - Add a simple, transparent spaced-repetition review flow.

5. **Study sessions and basic progress**
   - Record study sessions.
   - Show useful activity and performance summaries.
   - Avoid unsupported claims about mastery.

Learning paths and advanced analytics follow after these foundations are reliable.

## 8. User experience principles

- Keep navigation consistent with Odelis's hybrid model: major modules remain available in the persistent sidebar, while Learning has contextual navigation within its own workspace.
- Prefer clear hierarchy and calm layouts over dense dashboards.
- Make important actions understandable and reversible where practical.
- Provide empty states that explain what a feature does and how to begin.
- Keep subject and topic organization flexible; do not force every learning activity into a rigid hierarchy.
- Make data ownership and export options clear.

## 9. Out of scope for the first implementation

The following are deliberately deferred:

- Accounts and cloud synchronization
- Multi-device conflict resolution
- Social learning, sharing, and leaderboards
- AI-generated study materials
- Complex adaptive scheduling algorithms
- Rich media flashcards
- Highly configurable quiz engines
- Advanced learning-path recommendations

These may be evaluated later, after the core local-first experience is dependable.

## 10. Architectural decision summary

| Decision | Direction |
|---|---|
| Navigation | Hybrid: persistent module sidebar + contextual module navigation |
| Data architecture | Local-first; no account required |
| Initial persistence | Existing browser storage behind a shared service boundary |
| Data portability | Versioned export and validated import |
| First product priority | Learning |
| First Learning foundation | Shared data/storage, then subjects and topics |
| Future cloud support | Deferred until the local experience is established |

This document should be updated when implementation reveals a meaningful reason to change these decisions.
