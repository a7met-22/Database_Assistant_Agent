# Legal Ticket Decision Agent: LLM Routing with SQLite Persistence and Email Notification

An experimental **Agentic AI workflow** that reads a user's message about
a contract problem, asks an LLM to choose **one action from a fixed
list**, stores every decision in a SQLite database, and sends an email
notification through Gmail when a ticket is actually created.

The notebook demonstrates a ports-and-adapters (clean architecture)
approach to wiring a language model, a database, and an external
notification service around a single decision step, with an emphasis on
fixed action sets, injected dependencies, safe fallback behavior, and
secret handling.

> **Project:** Database Connection Agent (Legal Ticket Decision Engine)\
> **Notebook:** `Database_connection_Agent.ipynb`\
> **Domain:** Agentic AI · LLMs · Python · SQLite · SMTP Email ·
> Dependency Injection · Clean Architecture\
> **Environment:** Google Colab

------------------------------------------------------------------------

## 1. Project Overview

The goal is to let users describe a contract-related issue in **Arabic**
and have an AI decision engine route it to the right next step, while
the application code controls storage and notifications.

Instead of letting the model act directly, the workflow keeps the model
to a single job, choosing the next action, and surrounds it with
application-controlled steps:

1.  Load credentials from Colab secrets.
2.  Send the user's message to the LLM with a system prompt that defines
    five allowed actions.
3.  Parse the model's JSON answer into a typed `DecisionResult`.
4.  Fall back to human escalation if the answer cannot be interpreted.
5.  Save every decision as a row in SQLite.
6.  Send an email notification only when the action is `CREATE_TICKET`.
7.  Display all stored records in a DataFrame.

The allowed actions are:

-   `CREATE_TICKET`
-   `ASK_FOR_MORE_INFO`
-   `REJECT_REQUEST`
-   `FLAG_OUT_OF_SCOPE`
-   `ESCALATE_TO_HUMAN`

The central engineering principle is:

> **The model proposes a decision; the application decides what that
> decision is allowed to trigger, and falls back to a human when the
> output cannot be trusted.**

------------------------------------------------------------------------

## 2. Key Features

### LLM-based decision engine

-   Uses an OpenAI-compatible client pointed at the Hugging Face
    router.
-   Uses the model `Qwen/Qwen3-Coder-Next:novita`.
-   The system prompt restricts the model to contract issues, forbids
    legal advice, and requires a JSON-only reply with `action`,
    `confidence`, and `reasoning`.
-   The user message is passed to the model as a JSON-encoded string.

### Fixed action set

-   The `ActionType` enum defines the only valid actions.
-   Parsing converts the model's string into the enum, so an unknown
    action raises an error and triggers the fallback.

### Safe fallback

-   Any JSON, key, value, or attribute error during parsing returns
    `ESCALATE_TO_HUMAN` with confidence `0.0`.
-   The reasoning field records the first 200 characters of the raw
    output for inspection.

### Dependency injection

-   The domain defines three ports: `IDecisionEngine`,
    `ITicketRepository`, and `INotificationService`.
-   The LLM client, model name, and system prompt are injected into the
    engine, so they can be replaced or mocked in tests.

### Persistence

-   `SQLiteTicketRepository` creates the `tickets` table on first use.
-   Stores every decision with its action, confidence, reasoning, and
    timestamp, not only created tickets.
-   Supports retrieving all rows or filtering by action.

### Email notification

-   `EmailNotificationService` sends an HTML email over Gmail SMTP with
    SSL.
-   Uses a Gmail **App Password** rather than the account password.
-   Triggered only for `CREATE_TICKET`; failures are caught and
    reported without stopping the run.

### Secret management

-   Reads `HF_API_KEY`, `GMAIL_ADDRESS`, and `GMAIL_APP_PASSWORD` from
    Colab secrets, so no credentials appear in the notebook.

------------------------------------------------------------------------

## 3. Architecture

| Component | Responsibility |
|-----------|----------------|
| Secrets loader | Reads the API key and Gmail credentials from Colab secrets. |
| Domain entities | `ActionType`, `DecisionResult`, and `TicketRecord` as typed, immutable data objects. |
| Ports | `IDecisionEngine`, `ITicketRepository`, and `INotificationService` define the contracts. |
| `LLMDecisionEngine` | Calls the model, extracts the JSON, and builds a `DecisionResult` or the fallback. |
| System prompt | Defines the scope, the five actions, and the JSON output format. |
| `SQLiteTicketRepository` | Saves decisions and reads them back, optionally filtered by action. |
| `EmailNotificationService` | Builds and sends the HTML notification email. |
| Runner loop | Processes test messages, saves each decision, and conditionally sends email. |

### Action routing

| Action | Meaning | Saved | Email |
|--------|---------|-------|-------|
| `CREATE_TICKET` | Clear, specific contract issue that can be tracked. | Yes | Yes |
| `ASK_FOR_MORE_INFO` | Message is vague or missing essential details. | Yes | No |
| `REJECT_REQUEST` | Not a contract problem that can be handled. | Yes | No |
| `FLAG_OUT_OF_SCOPE` | Entirely outside contract issues (medical, criminal, and so on). | Yes | No |
| `ESCALATE_TO_HUMAN` | Real legal risk, sensitive case, or unparseable model output. | Yes | No |

### High-level workflow

``` text
        User message (Arabic)
                |
                v
        Build prompt messages
     (system prompt + JSON message)
                |
                v
           LLM via HF router
         (Qwen3-Coder-Next)
                |
                v
        Extract JSON from output
           /             \
      Parse OK         Parse fails
          |                |
          v                v
   DecisionResult   ESCALATE_TO_HUMAN
   (typed action)    (confidence 0.0)
          \                /
           v              v
          Save record to SQLite
                  |
                  v
        Action == CREATE_TICKET ?
             /           \
           Yes            No
            |              |
            v              v
     Send Gmail email   Continue
     (errors caught)
                  |
                  v
        Display all records (DataFrame)
```

------------------------------------------------------------------------

## 4. Technical Highlights

### Ports and adapters

The domain layer depends only on abstract interfaces. The LLM, the
database, and the email service are adapters that implement those
interfaces and are supplied from outside. Replacing SQLite with another
store, or Gmail with another channel, does not change the domain
contracts.

### Typed decisions

The model's text is converted into a frozen `DecisionResult` with an
enum action, a float confidence, and a reasoning string. Everything
downstream works with typed values instead of raw strings.

### Fail-safe parsing

The parser extracts the outermost JSON object with a regular expression
and validates the action against the enum. On any failure, the safest
route is taken: escalate to a human instead of guessing.

### Decision persistence for every outcome

Every decision is stored, including non-ticket outcomes, which creates a
complete log of what the engine decided for each message. The
`get_by_action` method makes it easy to retrieve a single category,
such as only created tickets.

### Targeted notifications

Email is sent only when a ticket is actually created, and a failed send
is reported as a warning so it cannot block saving or processing of
later messages.

### Secrets outside the code

Credentials come from Colab secrets, and the Gmail integration uses an
App Password, which can be revoked without changing the main account
password.

------------------------------------------------------------------------

## 5. Technology Stack

| Technology | Role |
|------------|------|
| Python | Core implementation language. |
| Google Colab | Notebook environment and secret storage. |
| `openai` client | OpenAI-compatible API client used with the Hugging Face router. |
| Hugging Face router | Hosted inference endpoint for the model. |
| Qwen3-Coder-Next (via Novita) | Language model used for decisions. |
| `dataclasses` / `enum` / `abc` / `typing` | Domain entities and interfaces. |
| `json` / `re` | Response extraction and parsing. |
| SQLite (`sqlite3`) | Persistent decision and ticket storage. |
| `smtplib` / `email.mime` | Gmail SMTP email delivery. |
| Pandas | Displaying stored records as a table. |

------------------------------------------------------------------------

## 6. Running the Notebook

### Requirements

The notebook is designed for Google Colab. It needs a Hugging Face API
key and a Gmail account with an App Password.

Before running:

-   Open the notebook in Google Colab.
-   Add three secrets using the key icon: `HF_API_KEY`,
    `GMAIL_ADDRESS`, and `GMAIL_APP_PASSWORD`, and enable notebook
    access to them.
-   Create a Gmail App Password (this requires 2-Step Verification on
    the account).
-   Make sure the `openai` package is installed in the runtime.

### Execution order

Run the notebook cells from top to bottom. The main stages are:

1.  Load secrets.
2.  Define the domain layer (entities, enums, ports).
3.  Define the LLM decision engine.
4.  Define the system prompt, create the client and engine, and set the
    test messages.
5.  Define the SQLite repository.
6.  Define the email notification service.
7.  Run the test messages: decide, save, notify, and display results.

### Example interaction

A message such as:

``` text
عندي عقد إيجار والمالك غيّر بند التأمين بدون إشعار، بدي اعرف حقوقي.
```

is expected to be routed to `CREATE_TICKET`, saved to the database, and
announced by email. A message such as a recipe request is expected to be
flagged as out of scope and saved without an email. The exact results
depend on the model's output.

------------------------------------------------------------------------

## 7. Configuration

Important settings are defined in a few cells so they can be reviewed
and adjusted without changing the structure.

Review these settings:

-   `SYSTEM_PROMPT`: scope, allowed actions, and output format.
-   Model name and base URL of the inference endpoint.
-   Database path (default `tickets.db`).
-   Notification recipient (defaults to the sender's own address).
-   Which actions trigger notifications (currently only
    `CREATE_TICKET`, with a commented option for escalations).
-   `test_messages`: the sample inputs used in the run cell.

Never hard-code API keys or passwords in the notebook or commit them to
a public repository. Keep them in Colab secrets or environment
variables.

------------------------------------------------------------------------

## 8. Security Considerations and Limitations

This notebook is an educational prototype, not a production-grade legal
intake system.

| Area | Current approach | Important limitation |
|------|------------------|----------------------|
| Prompt injection | System prompt defines scope and output format; the message is sent as a JSON string. | A prompt is not a boundary; a crafted message may still steer the model's decision. |
| Output parsing | Regex capture of the outermost braces, then enum validation. | Greedy matching can capture invalid spans; `confidence` is not range-checked and is not used for routing. |
| API errors | Parsing errors fall back to human escalation. | Network or API failures during the model call are not caught inside the engine. |
| Data privacy | Credentials are in Colab secrets. | Users' contract details are sent to a third-party model provider and stored in plain text in SQLite. |
| Email content | Message and reasoning are placed into an HTML template. | Values are not HTML-escaped, so untrusted text is inserted into the email body as-is. |
| Ticket identifier | The row id is returned by the repository. | The `TicketRecord` passed to the notifier is never updated with that id, so the email subject and table show `None` for the ticket number. |
| Timestamps | Created with `datetime.utcnow()`. | This call is deprecated in recent Python versions and produces naive UTC values. |
| Table semantics | One table for all decisions. | The table is named `tickets` but also holds non-ticket outcomes; filter by `action` to find real tickets. |
| Testing | Four sample messages with expected actions in comments. | No assertions, no mocked engine tests, and no sample for `REJECT_REQUEST`. |

### Recommendations before production use

-   Catch and handle API and network errors around the model call.
-   Validate the full response schema, including a bounded confidence.
-   Use the confidence value in routing, for example escalating below a
    threshold.
-   Escape all dynamic values before inserting them into HTML.
-   Return a ticket that includes the saved id before notifying.
-   Review data-protection requirements before sending client text to
    an external model provider.
-   Add authentication, access control, and retention rules for stored
    messages.
-   Add automated tests with a mocked engine and repository.

------------------------------------------------------------------------

## 9. Repository Structure

``` text
.
├── Database_connection_Agent.ipynb
└── README.md
```

The SQLite file `tickets.db` is created at runtime and should not be
committed, since it may contain user messages. Do not publish
credentials or stored user data.

------------------------------------------------------------------------

## 10. Possible Future Improvements

-   Implement the planned use-case layer that ties engine, repository,
    and notifier together, instead of a script loop.
-   Implement the follow-up loop for `ASK_FOR_MORE_INFO` so the system
    can collect the missing details and re-evaluate.
-   Add a low-confidence threshold that escalates to a human.
-   Retry or fall back gracefully when the model call fails.
-   Notify on `ESCALATE_TO_HUMAN` as well as on created tickets.
-   Store the ticket status separately from the raw decision and update
    it through its lifecycle.
-   Add unit tests with a mock `IDecisionEngine` and an in-memory
    repository.
-   Build a labeled evaluation set in Arabic and measure action
    accuracy per category.
-   Compare the current model with other models under the same
    evaluation protocol.

------------------------------------------------------------------------

## 11. Key Takeaway

This project demonstrates how an LLM can be placed inside a controlled
ticket-routing workflow: the model chooses one action from a fixed list,
while the surrounding application owns parsing, fallback, persistence,
and notifications.

The most important design lesson is that **reliable agentic systems
require more than a capable model or a carefully written prompt: they
need explicit contracts, safe fallbacks, and application-level control
over what each decision is allowed to trigger.**

------------------------------------------------------------------------

## Author

Add your name, GitHub profile, and portfolio link here.

## License

No license has been specified. Add a `LICENSE` file before presenting
the repository as an open-source project.
