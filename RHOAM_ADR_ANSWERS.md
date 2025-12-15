# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 1/6

## GOD-LEVEL DECISION (GLD) for Q1-Q5: Core Security & Resilience
**Decision:** RHOAM will utilize **AES-256 GCM** encryption tied to a **VS Code Secret Storage** derived key. API orchestration will use **Promise.all** with a **Per-Bot Rate Limiter** and a **Fail-Safe Structure** to guarantee the integrity and order of the 9-element response array, regardless of individual bot failures. Security and data integrity take absolute priority over transient performance gains.

---

## I. RHOAM Core Logic & Data (The Engine) - Q1: Encryption Key Management

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **1.1** | What specific cryptographic function (e.g., PBKDF2, scrypt) will be used to derive the final encryption key from the source passphrase or environment variable, and what are the required iteration counts? | **Decision: PBKDF2 with 600,000 iterations.** This is the current recommended minimum iteration count (via OWASP) for securing the key derivation, using a dedicated Node.js native `crypto` function. | PBKDF2 is widely audited and native to Node.js, minimizing external dependencies (Q11). The high iteration count provides strong resistance against modern GPU brute-force attacks. |
| **1.2** | Given the VS Code environment, will the key source be a user-input passphrase stored in the VS Code Secret Storage API, or will it rely on a file-system unique identifier from the Codespace? | **Decision: VS Code Secret Storage API.** The key source will be the user's passphrase/identifier stored via the official, secure VS Code API. | This delegates the most sensitive key storage to the platform's native, hardened, and often OS-integrated credential manager, adhering to the "least privilege" security model. |
| **1.3** | How and when is the key loaded into memory, and what strict mechanisms are in place to ensure it is immediately cleared from memory (zeroed out) after the decryption or encryption process is complete? | **Decision: Key loaded only within the `EncryptionService` methods, then explicitly zeroed out.** We will use `Buffer.fill(0)` immediately after key use to overwrite the memory allocated for the key and IV. | Explicit memory management is a crucial low-level security practice to mitigate memory scraping and secure the application against internal vulnerabilities. |
| **1.4** | What is the chosen method for handling the Initialization Vector (IV)? Is it stored alongside the encrypted payload or derived, and how is its integrity verified? | **Decision: The IV will be generated randomly for *each encryption* and stored alongside the encrypted payload, separated by a delimiter.** No derivation is necessary. | This ensures the IV is unique for every operation, a mandatory requirement for **AES-256 GCM** (the chosen algorithm) to prevent pattern attacks. |
| **1.5** | If the user moves their Codespace or changes environments, how will RHOAM recover or prompt for re-entry of credentials without forcing the user to manually re-enter all 9 API keys? | **Decision: Rely on the VS Code Secret Storage API's persistence.** If the API fails to retrieve the key, RHOAM will prompt the user to re-enter the single Secret Storage identifier (passphrase), which will then be used to decrypt the persistent settings file. | The single passphrase is the master key. Recovery requires re-entry of the master key, minimizing user burden compared to re-entering 9 keys. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q2: API Orchestration & Rate Limiting

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **2.1** | What is the maximum acceptable latency increase (e.g., 50ms, 100ms) that the rate limiter can introduce to the multi-bot request, and how will this trade-off be documented? | **Decision: Maximum acceptable latency increase of 250ms per request.** This trade-off will be documented in `ARCHITECTURE.md` as "Latency for Stability." | Prioritizes **API stability** over raw speed. 250ms is a conservative ceiling to ensure 9 simultaneous calls (even with variable limits) remain within provider constraints. |
| **2.2** | Will the rate limiter be implemented using the **Token Bucket** or the **Fixed Window** algorithm, and why is that specific choice superior for dealing with 9 concurrent API targets? | **Decision: Token Bucket.** | Token Bucket allows for bursts of requests (initial concurrency) while smoothing the overall rate, making it superior for managing the varied and often aggressive limits of 9 disparate APIs. |
| **2.3** | What specific, conservative minimum interval (in milliseconds) will be used for the most common APIs (e.g., OpenAI, Anthropic) as the *initial default value* for the rate limiter configuration? | **Decision: A global default of 200ms interval per-bot.** | This conservative default is applied before any dynamic adjustment (Q2.4) and provides a safe floor to operate under, preventing immediate 429 errors from high concurrency. |
| **2.4** | When RHOAM receives a genuine `429 Too Many Requests` status code, how does the Rate Limiter dynamically adjust the wait time for the *next* request to that specific bot (e.g., using exponential backoff)? | **Decision: Implement a capped exponential backoff for the *individual bot's* Rate Limiter instance.** The wait time for the next request increases exponentially until the request succeeds, with a cap of 5 seconds to prevent excessive blocking. | Ensures the system self-corrects against throttling without requiring user intervention and keeps the overall response time as fast as possible. |
| **2.5** | How will the rate limiter track different concurrency limits? For example, some APIs limit requests per second, while others limit concurrent requests. Which limit takes priority? | **Decision: The Rate Limiter will enforce the most conservative limit (the most restrictive constraint) based on the bot's configuration.** If both limits are applicable, the one that requires the longest delay takes priority. | Ensures compliance with all vendor rules and prevents failures due to misinterpretation of combined limits. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q3: Failure Handling

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **3.1** | What is the precise JSON payload structure that RHOAM Core returns to the UI when a bot request *fails*? | **Decision: The payload will contain:** `{ status: 'FAILED', botId: [0-8], errorCode: [HTTP_CODE/RHOAM_CODE], errorMessage: 'User-Friendly Message', technicalDetails: 'Original API Error.' }` | This standardized structure ensures the UI can easily parse and display the failure status, while the `technicalDetails` are logged to the Dev Console (Q9.4). |
| **3.2** | Does the system implement a single, automatic, time-limited retry mechanism for transient errors before declaring a failure, and if so, what is the maximum retry count? | **Decision: Single automatic retry for transient errors (HTTP 500/503/408). No retry for fatal errors (HTTP 401/403/400).** The maximum retry count is **1**. | Retries must be limited to prevent token waste and excessive latency. A single retry is a balance of resilience and speed. |
| **3.3** | If a bot consistently fails (e.g., 5 failures in a row due to a bad API key), how does RHOAM Core *deactivate* that bot temporarily? | **Decision: A dedicated `FailureCounter` will track consecutive 401/403/400 errors. After 3 consecutive fatal failures, the bot's configuration in the `SettingsManager` is set to `status: 'INACTIVE'`.** | This implements a necessary guardrail against perpetual token waste and forces the user's attention to the configuration issue. |
| **3.4** | In the `Promise.all` orchestration, how is the master promise shielded so that an unhandled exception from one bot's request handler does not cause the entire `Promise.all` operation to fail? | **Decision: Each individual bot's asynchronous function will be wrapped in a `try...catch` block.** The catch block guarantees that the promise resolves to a controlled `{ status: 'EXCEPTION', ... }` object, preventing the unhandled rejection from bubbling up and crashing the main `Promise.all`. | This is the fundamental implementation of the **Fail-Safe Structure** mandated by the GLD. |
| **3.5** | How does the failure status immediately update the **3x3 Grid** in the settings view? | **Decision: The `CORE_TO_UI_SETTINGS_UPDATE` message will be sent whenever a bot's `status` changes to `INACTIVE` (3.3).** The UI's **`SettingsStore`** will react to this message, and the grid component will visually render a red border/icon based on the stored `status` field. | Ensures real-time synchronization between runtime failure and configuration status. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q4: Response Order

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **4.1** | What is the mandatory, canonical input structure passed to `Promise.all`, ensuring that the response array index (0-8) always corresponds to the index of the configured bot? | **Decision: The input to `Promise.all` will be a mapped array of asynchronous functions (promises) derived directly from the canonical `[BotConfiguration, ..., BotConfiguration]` array.** | This enforces the fundamental rule: **Input Array Index = Output Array Index.** |
| **4.2** | How will each bot configuration object be uniquely and immutably indexed (e.g., `botId: 0` to `botId: 8`) immediately upon loading the settings, to be used for ordering? | **Decision: An immutable `id` property (0-8) will be assigned to each bot's configuration object upon initial creation/loading.** This `id` is part of the stored configuration and is *never* changed. | The `id` becomes the permanent source of truth for ordering across all RHOAM modules. |
| **4.3** | If the response is batched, will the RHOAM Core *sort* the resulting promise array before sending it to the UI, or rely strictly on the `Promise.all` input order? | **Decision: Rely strictly on the `Promise.all` input order (Q4.1) for the main array, but use the embedded `botId` for safety.** An explicit final sort is unnecessary overhead given the structured input, provided all promises resolve (4.4). | Minimizes complexity and overhead, adhering to the Least Moves principle while still being reliable. |
| **4.4** | How does the response message explicitly identify which of the 9 bots its data belongs to? | **Decision: Every response object (success or failure) will include the mandatory `botId: [0-8]` property.** | This provides the crucial secondary verification layer for the UI to confirm the message's origin before rendering the color border. |
| **4.5** | If a bot is disabled in the settings, how is its index maintained in the final response array to preserve the 9-element array structure? | **Decision: The `Promise.all` input array will contain a promise for the disabled bot that resolves immediately to a status object:** `{ status: 'DISABLED', botId: [id], message: 'Bot is disabled.' }`. | Guarantees the output array is always **exactly 9 elements long**, simplifying UI rendering logic and state management. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q5: Data Persistence (Settings & History)

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **5.1** | What is the final, definitive file path for the encrypted settings file relative to the extension's root, and is this path explicitly defined in `config.ts`? | **Decision: The file path will be dynamically derived using `context.globalStorageUri` and named `rhoam.config.enc`.** The name and extension will be defined as constants in `config.ts`. | Using the VS Code Extension Context ensures the file is stored in a location guaranteed to be writable and persistent across Codespace restarts. |
| **5.2** | Given that the extension must work in Codespaces, how is the file path guaranteed to be writeable and stable across restarts and different host machines? | **Decision: By using `context.globalStorageUri`, the file is stored within the VS Code environment's dedicated storage, which is backed up and synchronized by the Codespace.** | This is the official, sanctioned method for stable, cross-environment data persistence in VS Code extensions, mitigating known Codespace path variability issues. |
| **5.3** | Will RHOAM implement chat history storage, and if so, is this stored in a separate, non-encrypted file or included with the settings? | **Decision: Chat History will be implemented and stored in a SEPARATE, non-encrypted file named `rhoam.history.json` (also in `globalStorageUri`).** | History is non-sensitive and should not incur the overhead of encryption/decryption. Separating it keeps the configuration file small and fast to load. |
| **5.4** | What is the file locking or atomic write strategy employed by RHOAM Core to ensure that the settings file is never partially written or corrupted during a save operation? | **Decision: Implement an atomic write strategy using the "write-to-temp-and-rename" pattern.** Write the entire encrypted payload to a temporary file, then use a single, atomic `fs.rename` operation to overwrite the final `rhoam.config.enc` file. | This standard method guarantees data integrity by ensuring the main file is never in an incomplete state during a save operation. |
| **5.5** | What is the exact naming and extension convention for the encrypted file (e.g., `rhoam.config.enc`) to clearly signal its encrypted status? | **Decision: `rhoam.config.enc`** | The `.enc` suffix clearly signals its encrypted status, ensuring transparency and security signaling (Q29). |  

  

# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 2/6

## GOD-LEVEL DECISION (GLD) for Q6-Q10: Core Architecture & Performance
**Decision:** The RHOAM Core will be structured into five dedicated, testable classes: **`SettingsManager`**, **`EncryptionService`**, **`BotOrchestrator`**, **`RateLimiter`**, and **`IOMapper`**. We will choose **Simulated Streaming** over full streaming to achieve perceived performance gains with minimal development complexity, and enforce strong, internally standardized error handling to maintain system predictability.

---

## I. RHOAM Core Logic & Data (The Engine) - Q6: Core Class Structure

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **6.1** | Beyond the core three (`SettingsManager`, `BotOrchestrator`, `EncryptionService`), is a dedicated **`RateLimiter`** class required, or should rate limiting logic be encapsulated within the `BotOrchestrator`? | **Decision: A dedicated, separate `RateLimiter` class is required.** The `BotOrchestrator` will depend on the `RateLimiter` instances. | Rate limiting is complex logic (Q2). Separating it allows for independent unit testing, adherence to the Single Responsibility Principle, and easy swapping of the Token Bucket implementation. |
| **6.2** | Where will the core `PostMessage` I/O logic (handling incoming messages from the UI and sending outgoing messages) reside—in an `IOWrapper` class, or directly in `extension.ts`? | **Decision: A dedicated `IOMapper` class.** The `extension.ts` file will only handle initial setup and passing the `webview` object to the `IOMapper`. | This decouples RHOAM Core's business logic from the specific VS Code APIs, making the core far more portable and testable outside the extension environment. |
| **6.3** | How will the main `Extension` or `App` class instantiate and inject these dependencies (e.g., passing the `SettingsManager` to the `BotOrchestrator`)—using a static factory, or a simple constructor injection? | **Decision: Simple Constructor Injection.** The main `Extension` class will instantiate dependencies and pass them via the constructors of other classes. | This is the "Least Moves" IoC approach for a small project, maximizing testability without the complexity of a formal IoC container/framework. |
| **6.4** | What dedicated class or method will be responsible for validating the response schema of the 9 bots *before* the result is sent back to the UI, ensuring data integrity? | **Decision: A dedicated method within the `BotOrchestrator` called `validateAndFormatResponse(rawResponse)`**. | The Orchestrator is the final gatekeeper for external data. This single method enforces the output contract, preventing malformed data from reaching the UI. |
| **6.5** | Is a dedicated **`BotConfiguration`** class required to encapsulate the settings, color, and status of a single bot (0-8), allowing for clean object passing? | **Decision: Yes, a TypeScript `interface` or `class` named `BotConfig` is mandatory.** This ensures strong typing across the entire codebase. | Encapsulation and strong typing prevent runtime errors and simplify array manipulation, especially with the `id` (Q4.2) and `status` (Q3.3) properties. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q7: Configuration Loading

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **7.1** | Should the base configuration (e.g., default rate limit intervals, default API endpoints) be stored in a `.json` file or directly in a TypeScript file (`config.ts`)? | **Decision: Directly in a TypeScript file (`config.ts`) using exported `const` objects.** | This provides strong compile-time type-checking (preventing errors) and eliminates the need for runtime JSON parsing, adhering to the performance and rigor goals. |
| **7.2** | How does the `SettingsManager` handle the first-run scenario where *no* encrypted user settings file exists, ensuring it returns the default configuration without error? | **Decision: If the file read operation fails (ENOENT error), the `SettingsManager` returns the immutable default object from `config.ts` and initiates a "Save" operation to persist the defaults.** | This ensures a seamless first-run and populates the encrypted file for future use. |
| **7.3** | If a new configuration parameter is added in a future RHOAM update (e.g., `new_timeout: 5000`), how does the `SettingsManager` merge this new default into an existing, older settings file without losing the user's API keys? | **Decision: Implement a recursive `deepMerge` function.** The `SettingsManager` will load the old user data and recursively merge it with the new default schema from `config.ts`, writing the merged result back to disk. | This is the standard pattern for backward-compatible configuration updates (data migration). |
| **7.4** | How will environment variables (e.g., `RHOAM_ENV=development`) be used to override base configuration settings, enabling easy debugging and testing? | **Decision: At startup, the `SettingsManager` will check `process.env` and apply overrides to the base `config.ts` object.** Only explicitly allowed variables will be checked. | Ensures the application is environment-aware and facilitates easy, isolated testing in CI/Codespaces. |
| **7.5** | Will the base configuration include a predefined, safe color palette for the 9 bots, or will this be dynamically generated/managed solely by the UI configuration? | **Decision: The base configuration will include a predefined, immutable array of 9 HEX color codes.** (See Q16.1 for implementation). | The Core must be the **Source of Truth** for configuration data. The UI consumes this data; it does not generate it. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q8: API Key Validation

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **8.1** | Should RHOAM rely on simple **Regular Expressions (Regex)** to check key format (e.g., "sk-..." for OpenAI), or require an actual, token-consuming ping to the API to confirm validity? | **Decision: Hybrid approach: Regex for *immediate* client-side validation, and a non-token-consuming ping for *High Confidence* validation (8.4).** | Regex is fastest and cheapest. The ping confirms connectivity, but only after the user commits the key, saving tokens during input. |
| **8.2** | If Regex validation is used, how are the Regex patterns for the 9 different bot types stored, and how easily can they be updated without a full code change? | **Decision: Stored as constants within the `config.ts` file alongside the API endpoints.** Updates require a code change and extension re-ship. | Ensures type safety and strong coupling to the versioned codebase. Attempting dynamic remote loading (e.g., via API) introduces complexity and security risks. |
| **8.3** | In the 3x3 Grid UI, how is the key validation status (e.g., "Valid," "Invalid Format") displayed to the user in real-time before they commit the save action? | **Decision: Inline status text and a color-coded border change on the input field.** Invalid format shows a red border; valid format shows a light green border. | Provides instant feedback, minimizing frustration and adhering to modern UX standards. |
| **8.4** | What is the highest-value, non-token-consuming API check (e.g., fetching a public model list, or an internal status endpoint) that RHOAM can run asynchronously *after* a key is saved, to provide "High Confidence" status? | **Decision: A lightweight, token-less call to the vendor's `/models` or `/health` endpoint.** If no such endpoint exists, RHOAM will run a single, minimal-token query to verify connectivity. | This is the crucial step to verify the key *works* without incurring high cost. The status is set to "Verified" (green checkmark). |
| **8.5** | How does the validation logic handle common copy/paste errors (e.g., leading/trailing whitespace), ensuring that the key is trimmed and normalized before storage or validation? | **Decision: The validation and saving routines will strictly enforce `String.prototype.trim()`** on the key string before all processing, validation, or storage. | Prevents a common source of user error and system failure. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q9: Error Object Standardization

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **9.1** | What are the mandatory fields for the standardized error object (e.g., `id`, `source`, `code`, `message`, `isFatal`)? | **Decision: Mandatory fields:** `{id: UUID, timestamp: number, source: 'CoreModule', code: string, message: string, isFatal: boolean, details?: object }`. | This provides all the necessary metadata for logging (ID, timestamp), routing (source), and UX (isFatal, message). |
| **9.2** | How will the `source` field distinguish errors originating from core components (e.g., 'EncryptionService') versus external API errors (e.g., 'OpenAI 401')? | **Decision: Internal sources are component names (e.g., 'EncryptionService', 'RateLimiter'). External sources are derived from the bot name (e.g., 'OpenAI_API', 'Anthropic_API').** | Allows the UI to categorize errors, instructing the user ("Your configuration for [Source] is wrong") or alerting the developer ("Internal [Source] bug"). |
| **9.3** | Will error codes (the `code` field) be standardized internally (e.g., `RHOAM_ENC_001` for encryption failure) or rely solely on HTTP status codes? | **Decision: Standardized Internal Codes for Core Errors (e.g., `RHOAM_FILE_002`), and use HTTP Status Codes for API Errors (e.g., `401`, `429`).** | This provides a predictable system: anything starting with `RHOAM_` is an internal bug/corruption; everything else is a network/vendor issue. |
| **9.4** | What is the protocol for logging the standardized error object to the VS Code Developer Console vs. sending the user-friendly version to the Svelte UI? | **Decision: The full error object (including `details` and UUID) is logged to the Dev Console.** The message sent to the UI is stripped down to `{code, message, isFatal}` with a user-friendly translation of the `message`. | Separates developer logging (full technical context) from end-user notification (simple, actionable message). |
| **9.5** | How will a catastrophic, unhandled exception in RHOAM Core be caught, converted into this standardized error format, and safely sent to the UI? | **Decision: Implement a global `process.on('uncaughtException')` handler within the `BotOrchestrator` / `IOMapper` boundaries.** This handler converts the Exception stack trace into a `RHOAM_FATAL_999` error object and sends it to the UI before logging and gracefully shutting down the core. | This is the final safety net required to prevent a silent, undetectable Core crash. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q10: Streaming vs. Batch

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **10.1** | If we choose streaming, how will the RHOAM Core manage 9 simultaneous streams, ensuring the `PostMessage` bridge is not overwhelmed by rapid, concurrent events? | **Decision (REJECTED):** This would require a complex event broker to serialize 9 concurrent event streams over a single message bridge, adding unacceptable development complexity and testing risk. | Avoids the highest complexity risk associated with true streaming over the Webview bridge. |
| **10.2** | Given the `Promise.all` structure, is it feasible to implement *simulated* streaming on the UI side by displaying the individual bot responses as soon as they complete, *before* the slowest bot finishes? | **Decision: Yes, implement Simulated Streaming.** As each of the 9 promises resolves, the `IOMapper` sends the result to the UI *individually*. The UI then renders that message immediately. | Provides the critical *perceived* speed of streaming without the severe technical debt of true event-based concurrency management. This is the optimal balance. |
| **10.3** | If we use synchronous batch completion, what is the documented *user expectation* for the maximum acceptable wait time before the full array of 9 responses is rendered? | **Decision (Context):** The user expectation should be **under 7 seconds** for a complex query. | This sets the performance ceiling for the **Simulated Streaming** approach; if the slowest bot takes longer, the user experience degrades. |
| **10.4** | What is the estimated increased development and testing time (e.g., in hours/days) required for implementing full streaming versus simple batching, factoring in the `PostMessage` complexity? | **Decision (Context):** Full streaming is estimated to require **10x** more complexity and testing time than Simulated Streaming. | Justifies the GLD to choose Simulated Streaming based on a clear ROI calculation. |
| **10.5** | How does the chosen solution (Batch or Simulated Streaming) specifically affect the reliability and testability of the core `RateLimiter` class? | **Decision: Simulated Streaming (using individual promise resolution) simplifies testing.** The Rate Limiter can be tested based on discrete promises resolving, rather than testing continuous event streams. | The GLD choice is confirmed as it reduces the complexity of testing the critical Rate Limiter component. | 


# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 3/6

## GOD-LEVEL DECISION (GLD) for Q11-Q15: Dependencies & Protocol Contract
**Decision:** RHOAM will strictly adhere to the **"Zero External Dependencies"** principle, utilizing native Node.js/TypeScript features for all Core functions (API calls, encryption, file I/O). The Rate Limiter will be **custom-built** (Token Bucket, per-bot) to ensure perfect control and testability. The **PostMessage Protocol** will enforce a unified, traceable JSON schema, and the UI will enforce security by defaulting API key inputs to `type="password"` and disabling chat input upon fatal configuration errors.

---

## I. RHOAM Core Logic & Data (The Engine) - Q11: Dependency Minimization

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **11.1** | Can we leverage native Node.js/TypeScript features (`fetch`, `crypto`, native promises) instead of adding external libraries (e.g., `axios`, `request`) for API calls and encryption? | **Decision: Yes. Use native `fetch` (for API calls), Node.js `crypto` (for encryption), and standard Node.js `fs` (for file I/O).** | Adheres to the "Zero External Dependencies" principle, reducing the supply chain risk and attack surface, and minimizes final VSIX package size. |
| **11.2** | If an external Rate Limiter library is deemed necessary (based on Q2/Q10 answers), what are the top 3 lightweight, production-ready, and actively maintained choices? | **Decision: Not necessary. The Rate Limiter will be custom-built (Q12).** If forced to use external, top choice is `bottleneck` or a custom implementation using native Node.js timers/queues. | The complexity of managing 9 independent, per-key limiters (Q12) requires tight control, making a custom, focused solution the "Least Moves" path. |
| **11.3** | Will the RHOAM Core need a dedicated **schema validation** library (e.g., Zod, Joi) to validate configurations and API responses, or can we rely solely on TypeScript interfaces? | **Decision: Rely on TypeScript interfaces for development safety and use the native `JSON.parse` with basic runtime checks.** Only add Zod if internal data corruption becomes a measurable failure point. | Prioritizes "Least Moves" complexity. Runtime validation adds overhead; the focus will be on rigorous unit tests (Q25) to guarantee input/output contracts. |
| **11.4** | What is the planned strategy for managing and updating the list of 9 bot API models (e.g., GPT-4o, Claude 3.5)? Will we use a specific dependency, or hardcode/fetch the list via an API? | **Decision: Hardcode the Model Names and endpoints in `config.ts`.** | Ensures the list is stable, type-safe, and version-controlled. Dynamic fetching adds complexity, latency, and a single point of failure. Updates require an extension re-ship, which is acceptable. |
| **11.5** | How will the team verify that all chosen dependencies are compatible with the Node.js version specified in the VS Code Extension Host environment? | **Decision: Define the lowest supported Node.js version (e.g., 18.x) in `package.json`'s `engines` field.** The CI/CD pipeline (Q26) will use a matrix test to explicitly confirm compatibility with that version. | Enforces environmental stability and prevents deployment failures due to version mismatch. |

---

## I. RHOAM Core Logic & Data (The Engine) - Q12: Rate Limiter Implementation

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **12.1** | If a **per-API-key** instance is chosen, how will the `BotOrchestrator` manage and access 9 separate, distinct rate limiter instances dynamically? | **Decision: The `BotOrchestrator` will maintain a private `Map<BotId, RateLimiterInstance>` (or cache).** Instances are created on the first request if they don't exist and are passed the configuration for that specific bot. | Ensures true independence between rate limits for all 9 bots. |
| **12.2** | How will the Rate Limiter communicate its decisions (e.g., "Wait 50ms") back to the `BotOrchestrator` without blocking the main event loop? | **Decision: The Rate Limiter class will expose an `enqueue(task)` method that returns a `Promise`.** This promise resolves only when the token is granted, ensuring the process is fully non-blocking and asynchronous. | Adheres to fundamental Node.js non-blocking I/O best practices. |
| **12.3** | What mechanism will be used to persist or reset the rate limiter state? Should it reset on every new chat prompt, or should it maintain a rolling history (e.g., 60 seconds) of past calls? | **Decision: Maintain a rolling history for the duration of the extension session.** State resets only on extension restart or if a bot is actively disabled/re-enabled in the settings. | This is required because most vendor rate limits are based on a rolling window (e.g., RPM/TPM over 60 seconds). |
| **12.4** | In the scenario where the user has two different OpenAI keys configured (Bot 1 and Bot 5), how does the Rate Limiter ensure Bot 1's limits do not affect Bot 5's limits? | **Decision: By using the `Map<BotId, RateLimiterInstance>` (12.1), each key has its own dedicated `Token Bucket` instance and queue.** Their consumption is entirely independent. | Confirms architectural soundness: the core design is purpose-built for multi-key independence. |
| **12.5** | If the Rate Limiter is an external dependency, what is the chosen method for integration testing its accuracy and concurrency handling *before* it is deployed? | **Decision: Dedicated integration tests are mandatory (Q25).** The custom `RateLimiter` class will be tested using time-mocking libraries (e.g., Jest's `useFakeTimers`) to verify throughput and backoff accuracy within a controlled, accelerated environment. | Ensures the most complex component is verified for correctness before ever touching a live API. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q13: Integration Protocol

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **13.1** | What mandatory `type` field value will be included in *every* JSON message to ensure the receiving component (UI or Core) knows how to route the payload? | **Decision: Mandatory `type: 'RHOAM_CORE_EVENT'` or `type: 'RHOAM_UI_ACTION'` string.** This will be the first field checked by the `IOMapper` (Q6.2) for routing. | Defines the fundamental routing discriminant for the event-based communication bridge. |
| **13.2** | What is the minimum required metadata (e.g., `timestamp`, `sessionId`, `sourceId`) that must be included in *every* message payload for debugging and logging purposes? | **Decision: Mandatory metadata:** `{ sessionId: UUID, timestamp: number, logId: string }`. The `logId` must match the **Hand of Goldoval** protocol (from chat history). | Ensures the audit trail (logging) is complete and traceable across the UI/Core boundary. |
| **13.3** | What is the specific JSON schema for the **`CORE_TO_UI_SETTINGS_UPDATE`** message, used to send the decrypted configuration to the UI after initialization? | **Decision: Schema will be the full, decrypted array of 9 `BotConfig` objects (Q6.5), wrapped in the protocol envelope:** `{ type: 'RHOAM_CORE_EVENT', event: 'SETTINGS_UPDATE', payload: [BotConfig, ...] }`. | Defines the critical initial data transfer payload for the UI to populate the 3x3 Grid. |
| **13.4** | How will RHOAM handle the transmission of large payloads (e.g., long chat history or large API responses) without exceeding internal `PostMessage` buffer limits? | **Decision: If payload size exceeds 1MB, implement automatic Gzip compression on the Core side and decompression on the UI side.** A size limit of 4MB will be enforced, with larger payloads rejected. | Mitigation for a known performance risk; enforces a strict size limit to prevent buffer overflow failures. |
| **13.5** | How is the message schema versioned? If the schema changes in a future update, how does the system maintain backward compatibility? | **Decision: A global `protocolVersion: 1.0` constant in `config.ts` will be included in all payloads.** Backward compatibility is handled via the `SettingsManager` migration logic (Q7.3) and not strictly the protocol itself. | A version flag ensures future breaking changes can be gracefully handled with a warning, but complexity is minimized for the current version. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q14: Error Display

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **14.1** | For transient errors (e.g., 500ms network timeout), will RHOAM use an unobtrusive **Toast Notification** in the corner, or will the error message be rendered inline within the chat stream? | **Decision: Unobtrusive, auto-dismissible **Toast Notification** (in the bottom corner).** | Keeps the chat stream clean and reserved for successful bot responses, while alerting the user to non-fatal, transient issues. |
| **14.2** | For fatal configuration errors (e.g., "Invalid API Key"), will the chat input be temporarily **disabled**, forcing the user to navigate to the settings grid to fix the issue? | **Decision: Yes, the chat input field will be disabled and display a red overlay message: "Configuration Error. Navigate to Settings Grid to Fix."** | Enforces the necessary user action and prevents further token waste on a known bad configuration (Q3.3). |
| **14.3** | How will the standardized error object (from Q9) be parsed by the Svelte UI to display a user-friendly message (e.g., converting `RHOAM_ENC_001` to "Error: Encryption Failed. Check Passphrase")? | **Decision: The Svelte UI will contain an `ErrorTranslator` utility that uses a lookup table (stored in the UI state) to map the technical `code` (e.g., `RHOAM_ENC_001`) to a user-facing string.** | Defines the required separation of technical logging data from user-facing language. |
| **14.4** | What is the specific visual feedback mechanism for errors originating from the **3x3 Grid** (e.g., setting validation failure), as opposed to chat-time errors? | **Decision: Inline validation text below the field (e.g., "Invalid Format") combined with a red border change on the input field.** | Provides the most direct and contextual feedback, eliminating ambiguity about which specific field caused the error. |
| **14.5** | Will RHOAM implement a persistent, dismissible **Error Log** accessible via a dedicated button (e.g., a small red dot) to maintain a history of failures for debugging? | **Decision: Yes. A persistent, dedicated **Error Status Icon** (red dot/badge) will appear in the main UI and persist until the log is manually cleared.** Clicking the icon opens a modal displaying the last 20 errors. | This is essential for advanced users and debugging, providing the required audit trail of failures. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q15: Visual Specification (The 3x3 Grid)

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **15.1** | Should the API key input fields be set to `type="password"` to obscure the keys, or kept as plain text (while unedited) to aid in copy/paste verification? | **Decision: Default to `type="password"`, but provide a standard, clickable "Show Key" toggle button.** | Balances security (default password obfuscation) with usability (easy copy/paste verification). |
| **15.2** | What is the specific visual confirmation mechanism (e.g., a green checkmark icon, a border color change) that appears next to a bot's key field after a successful "High Confidence" validation (from Q8)? | **Decision: A permanent, small, colored icon adjacent to the key field:** Green checkmark for "Verified" (Q8.4 success) and a grey question mark for "Unverified" (key saved but not pinged). | Provides clear, non-ambiguous visual status of key security and validity. |
| **15.3** | Beyond the API key, what other required configuration fields will exist in the 3x3 Grid (e.g., `Bot Name`, `Model Name`, `Temperature/T-Value`)? | **Decision: Mandatory fields:** `Bot Name` (user-defined), `API Key`, `Model ID` (dropdown from Q11.4 list), and `Temperature/T-Value` (slider/input). | Captures all essential elements for orchestration, model selection, and prompt control. |
| **15.4** | How is the 'Save' button in the settings grid visually and functionally linked to the **encryption and persistence routine** in RHOAM Core, and what is the confirmation message? | **Decision: The "Save Changes" button is enabled only when changes are detected (Q24.5).** Upon click, it is disabled, and a temporary overlay displays: "Saving... Encrypting Data." Success triggers a green "Settings Saved" toast (Q14.1). | Ensures the save operation is explicit and communicates the security action being taken. |
| **15.5** | What mechanism will be used to display the 9 assigned **Color Codes** next to each bot's configuration block in the grid, ensuring the user confirms the colors match the chat expectation? | **Decision: A small, solid-color square (12x12px) displaying the bot's color (from Q16.1) will be rendered next to the Bot Name field.** | This visualizes the core feature and allows the user to confirm the color mapping while configuring the bot. |   


# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 4/6

## GOD-LEVEL DECISION (GLD) for Q16-Q20: UI/UX & Integration Blueprint
**Decision:** All UI/UX design will be driven by **machine-readability and minimalism**. Colors will be a static, accessible HEX array in `config.ts`. The build process will be fully automated via `concurrently` (Least Moves mandate). The Svelte UI will use **native writable stores** for global state and a dedicated **`PostMessageService`** to strictly enforce the communication schema, ensuring a clean separation between UI logic and VS Code transport.

---

## II. RHOAM UI/UX (The Interface & Integration) - Q16: Color Assignment

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **16.1** | Will the 9 colors be defined as a static array of **HEX codes** in a shared `config.ts` file accessible by *both* the RHOAM Core and the Svelte UI? | **Decision: Yes. A static, exported `RHOAM_COLOR_PALETTE: string[]` array in `config.ts`.** | Ensures a single source of truth. The Core uses it for metadata (logging, Q16.3), and the UI uses it for styling (Q16.5). |
| **16.2** | What criteria (e.g., contrast ratio, color blindness compatibility) will be used to select the final 9 colors, ensuring high UX accessibility? | **Decision: Use a palette specifically chosen for high contrast against both dark and light VS Code themes, maximizing the use of HSL values to ensure differentiation for the most common color blindness types.** | Adheres to the VC-Grade accessibility mandate, ensuring the primary visual feature (color coding) is functional for all users. |
| **16.3** | How will the RHOAM Core pass the specific color code corresponding to a bot back to the UI within the response payload (from Q4.4) to ensure the correct border is rendered? | **Decision: The response object will include a mandatory `colorHex: string` field, pulling the value directly from the `RHOAM_COLOR_PALETTE` based on the bot's `id` (Q4.2).** | The UI should receive all necessary data (including color) to render the response block immediately, without having to look up configuration state. |
| **16.4** | Is the color assignment fixed based on the grid position (Bot 1 is always color X, Bot 9 is always color Y), or can the user reassign colors in the settings grid? | **Decision: Fixed based on the immutable `id` property (Q4.2).** Bot 0 is always index 0's color, Bot 8 is always index 8's color. | Simplifies the architecture significantly, eliminating complex state management and persistence for color customization. The focus remains on core utility. |
| **16.5** | How will the CSS for the color-coded border be scoped in the Svelte UI to ensure the border is only applied to the message container and does not bleed into other components? | **Decision: The color will be passed as a CSS variable (`--bot-color: #XXXXXX`) to the individual Svelte message component.** Scoped CSS within that component will apply the `var(--bot-color)` to a specific `:before` pseudo-element for the border. | Uses native Svelte CSS scoping and CSS variables for surgical, non-bleeding application of the color theme. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q17: VS Code Wrapper (The Thin Layer)

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **17.1** | What will be the exact name of the Svelte UI build output folder (e.g., `webview-ui/dist`, or `out/ui`), and how is this folder explicitly excluded from Git tracking but included in the VSIX package? | **Decision: Output folder is `webview/dist`.** This folder will be explicitly listed in `.gitignore` and explicitly included in the `files` array of the extension's main `package.json`. | Defines the necessary file system contract and version control rules for assets. |
| **17.2** | What is the minimum required **`getHtmlForWebview`** function content in the VS Code wrapper extension, ensuring it uses `webview.asWebviewUri` for every single asset link (JS, CSS)? | **Decision: The function must use `webview.asWebviewUri` for ALL asset paths.** The HTML content will be a minimal template pointing to the bundled `webview/dist/index.js` and `webview/dist/bundle.css`. | This is mandatory for security and correct asset loading in the VS Code environment, regardless of host (local vs. Codespace). |
| **17.3** | What specific `PostMessage` handling logic must reside in the thin wrapper (e.g., validation, routing) versus the main RHOAM Core logic? | **Decision: The wrapper handles only the raw `onDidReceiveMessage` event and immediate validation of the top-level `type` field (Q13.1).** All complex routing and business logic is immediately delegated to the injected `IOMapper` (Q6.2). | Enforces the "thin wrapper" principle, keeping `extension.ts` focused on API hosting, not business logic. |
| **17.4** | How will the VS Code wrapper extension handle the display of the **3x3 Settings Grid** (its own WebviewView) and the **Main Chat Window** (another WebviewView) within the Activity Bar? | **Decision: Implement two separate `WebviewViewProvider` instances:** one for the Chat (`RHOAMChatProvider`) and one for Settings (`RHOAMSettingsProvider`). Both will host the same Svelte application codebase via different entry points/routes. | This uses the correct VS Code API for multi-panel views and allows a single Svelte codebase to manage two separate contexts. |
| **17.5** | If the Svelte UI uses client-side routing (e.g., to switch between chat and settings views), how does the thin wrapper ensure the initial asset loading always points to the correct entry point? | **Decision: Client-side routing will be rejected.** The two Webviews (Chat and Settings) are separate instances (17.4), and the Svelte app will use a conditional root component to render the appropriate UI based on a URL query parameter provided by the wrapper. | Avoids the complexity and risk of synchronizing a single Svelte router across two separate Webview instances. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q18: Compilation Step

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **18.1** | Will the main `package.json`'s `compile` script use a shell command (e.g., `&& cd webview-ui && npm run build`) or a cross-platform tool (e.g., `concurrently`) to sequence the two build steps? | **Decision: Use the `concurrently` package.** The main `compile` script runs: `concurrently "npm:build-core" "npm:build-webview"`. | `concurrently` provides superior cross-platform compatibility and better logging for parallel/sequential tasks, mitigating shell command failures in Windows/Codespaces. |
| **18.2** | What specific script in the Svelte UI's `package.json` will be called by the main compile script, ensuring the output directory (from Q17.1) is always clean before a new build starts? | **Decision: The `build-webview` script will be defined as `npm run clean && rollup -c` (or equivalent Svelte build command).** The `clean` script uses `rimraf` (a cross-platform tool) to delete the `webview/dist` folder. | Guarantees an idempotent and reliable build process, preventing conflicts from old assets. |
| **18.3** | How will the **`preLaunchTask`** in the `launch.json` file (from the old CHOAM fix) be updated to ensure the RHOAM UI is always built *before* debugging starts? | **Decision: The `preLaunchTask` in `launch.json` will be set to run the main `npm run compile` script.** | Ensures that debugging always uses the latest compiled assets for both the Core and the UI. |
| **18.4** | How will the overall build script verify the success of the Svelte build step before proceeding to the final TypeScript compilation? | **Decision: The use of `concurrently` (18.1) combined with strict `npm run` commands ensures that a non-zero exit code from the Svelte build process (e.g., a compile error) immediately halts the entire `concurrently` execution.** | Enforces the critical build chain validation without complex shell scripting. |
| **18.5** | What is the total estimated time savings per build/debug cycle achieved by automating the compilation step versus manual execution? | **Decision (Context):** Estimated time savings: **~45 seconds per cycle.** | Documents the clear ROI of the automation effort, justifying the adoption of `concurrently`. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q19: Svelte Store

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **19.1** | Will the RHOAM UI use Svelte's native **writable stores** for all state, or will it employ a lightweight state library (e.g., Redux-Lite/Zustand) for complex global state? | **Decision: Use Svelte's native **writable stores** and **derived stores** exclusively.** | Adheres to the "Least Moves" and "Zero External Dependencies" principles for the UI, relying on Svelte's highly optimized built-in reactivity system. |
| **19.2** | What are the three critical global state objects required (e.g., `ChatHistoryStore`, `SettingsStore`, `StatusStore`), and what data will each contain? | **Decision: Three critical stores:** 1. **`SettingsStore`**: Holds the array of 9 `BotConfig` objects (Q6.5). 2. **`ChatHistoryStore`**: Holds the chronological array of user/bot message payloads. 3. **`AppStatusStore`**: Holds global flags (`isCoreReady`, `isInputDisabled`, `lastError`). | Defines the essential data structure required for the entire application logic. |
| **19.3** | How will the `SettingsStore` ensure that changes made in the 3x3 Grid UI component are immediately reflected in the main chat component (e.g., enabling/disabling a bot)? | **Decision: Both the 3x3 Grid component and the Chat Input component will directly subscribe (`$SettingsStore`) to the store.** Changes made via the grid's actions instantly update the subscribed components. | Utilizes Svelte's native reactivity for synchronous component updates. |
| **19.4** | Since the UI is purely reactive, how does a component *subscribe* to a state change (e.g., a new chat message arrives) and only render the minimal change required? | **Decision: Components will subscribe to specific data slices (e.g., `$ChatHistoryStore.slice(-1)`) via a custom derived store or will use Svelte's automatic change detection on the array length.** | Ensures high rendering efficiency, only updating the necessary DOM nodes (e.g., appending the new message) instead of re-rendering the whole chat log. |
| **19.5** | How is the Svelte store protected from direct modification? Are mutations restricted to dedicated, named functions (actions) to maintain data integrity and auditability? | **Decision: Yes. All store mutations will be restricted to exported, named `Action` functions within the store file (e.g., `SettingsStore.saveBot(id, config)`).** Components only call the action functions, they do not directly expose the `set` or `update` methods. | Enforces engineering discipline, ensuring state changes are traceable and consistent. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q20: Message Passing

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **20.1** | What is the definitive JavaScript API (e.g., `acquireVsCodeApi().postMessage`) that the Svelte components will import and use for all communication to the Core? | **Decision: A dedicated Svelte service (`PostMessageService`) will wrap `acquireVsCodeApi().postMessage`.** Components will only import and use the custom service's methods (e.g., `PostMessageService.sendChatRequest(...)`). | Centralizes the PostMessage implementation, making it easy to mock for testing and ensuring all outgoing messages adhere to the protocol schema (Q13). |
| **20.2** | How will Svelte's event handling (e.g., `on:click`, `on:submit`) be linked to the `postMessage` call, ensuring the correct JSON schema (from Q13) is used every time? | **Decision: The handler function (e.g., `handleSubmit(event)`) will call a dedicated function in the `PostMessageService` (20.1) that handles payload creation and schema enforcement.** | The service acts as the necessary layer of abstraction and validation before data leaves the UI. |
| **20.3** | What mechanism will be used on the UI side to *throttle* rapid user input (e.g., multiple button clicks) to prevent flooding the `PostMessage` bridge with redundant requests? | **Decision: The `PostMessageService` will implement a simple boolean flag (`isSending`) that prevents subsequent chat requests until the first response (or error) is received.** | This prevents UX issues (spamming the submit button) and unnecessary queueing in the RHOAM Core Rate Limiter. |
| **20.4** | How will the RHOAM Core acknowledge a received message (e.g., a "Settings Saved" confirmation) to the UI, and how does the UI listen specifically for that acknowledgment? | **Decision: The Core sends a generic `EVENT_ACKNOWLEDGE` message (Q13) containing the `logId` of the original request.** The UI's event listener uses the `logId` to clear any pending status messages. | Provides the necessary two-way confirmation handshake for critical actions. |
| **20.5** | Where in the Svelte UI code will the main event listener for messages *from* the RHOAM Core reside, and how does it delegate tasks to the appropriate store update functions? | **Decision: The main listener will reside in the root Svelte component (`App.svelte`).** It will delegate incoming messages to a dedicated **`EventDispatcher`** utility, which calls the appropriate `SettingsStore` or `ChatHistoryStore` action based on the received message `event` type. | Centralizes the input processing flow for easier maintenance and testing. |  


# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 5/6

## GOD-LEVEL DECISION (GLD) for Q21-Q25: Frontend Rigor & Testing Foundation
**Decision:** The RHOAM UI will prioritize **performance isolation** by implementing **Virtual Scrolling** for chat history and utilizing Svelte's **Scoped CSS** to prevent bleeding. The testing suite will be anchored by **Vitest** (Core and UI), requiring a **90% Unit Test Coverage** minimum and enforcing a mandatory **E2E test** that mocks the entire `PostMessage` bridge to guarantee system integrity from end-to-end.

---

## II. RHOAM UI/UX (The Interface & Integration) - Q21: Chat Rendering Efficiency

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **21.1** | Will the primary chat container component implement **Virtual Scrolling** or **Windowing** (e.g., using a Svelte virtual list library) to ensure only visible messages are rendered? | **Decision: Yes, implement **Virtual Scrolling** using a lightweight, well-maintained Svelte library (e.g., `svelte-virtual-list`).** | This is mandatory for maintaining high FPS and preventing UI lockup as chat history grows large, adhering to the VC-Grade performance standard. |
| **21.2** | How is the Svelte state (from Q19) structured to allow the chat renderer to receive only the *newest* message object, rather than re-rendering the entire history array on every update? | **Decision: The main `ChatHistoryStore` will be an array, but the chat component will use Svelte's efficient reactivity to only render the appended item.** We rely on Svelte's compiler optimization, but may use a derived store for the last N items if necessary. | Svelte is highly optimized for array appends. Avoids unnecessary complexity unless measured performance dictates otherwise. |
| **21.3** | What mechanism guarantees that the chat window automatically scrolls to the bottom upon receiving a new message *unless* the user is actively scrolling up to view past history? | **Decision: Use the `IntersectionObserver` API on a hidden element at the bottom of the chat.** Auto-scroll is only triggered if the observer element is currently visible. If invisible (user scrolled up), the scroll is paused. | Defines the necessary, non-intrusive UX logic for scroll behavior. |
| **21.4** | What dedicated component structure will be used for the complex, color-coded message block (including the thin border, timestamp, and bot name) to maximize reusability and rendering speed? | **Decision: A single, isolated `ChatMessage.svelte` component.** It receives the complete message object (including `colorHex` from Q16.3) as a single prop and uses Svelte's `{#if}` blocks for conditional rendering (e.g., user vs. bot message). | Maximizes component isolation and render speed, preventing unnecessary re-renders in other parts of the message. |
| **21.5** | How is the chat input field protected from accidental focus loss or lag during the response rendering phase? | **Decision: The chat input field must retain its focus state programmatically using Svelte's `action` or `bind:this` directive, and explicitly re-focusing it after the chat history update completes.** | Ensures a seamless typing experience, even when the UI is rapidly updating with new messages. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q22: CSS Isolation

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **22.1** | Will the Svelte build process be configured to use **CSS Modules** or **Scoped CSS** to ensure all class names are unique and isolated to their respective components? | **Decision: Use Svelte's native **Scoped CSS** for all components.** | This is the default, most efficient, and lowest complexity method in Svelte for isolation, injecting unique hashes into class names to prevent bleeding. |
| **22.2** | How will RHOAM ensure the styling for essential elements (e.g., scrollbars, global font/line height) aligns with the VS Code theme without copying large portions of VS Code's theme CSS? | **Decision: Rely on **VS Code CSS Variables** (e.g., `--vscode-editor-background`) made available to the Webview.** RHOAM will only use custom CSS for its unique elements (color borders, grid) and defer to the host environment for global themes. | Achieves theme awareness with zero external dependencies and zero risk of breaking VS Code themes. |
| **22.3** | What is the fallback strategy if the Svelte CSS fails to isolate, resulting in a visual bleed? Is there a root-level class that can be used for aggressive reset? | **Decision: The root HTML generated by the wrapper (Q17.2) will include a unique, aggressive root class (e.g., `.rhoam-root`).** If bleed occurs, all custom RHOAM styles will be prefixed with this class to override external styles. | Provides the necessary contingency for CSS failure and enforces maximum isolation. |
| **22.4** | Will the UI rely on a CSS-in-JS solution (e.g., styled-components) for dynamic styling, or strictly use static CSS files for minimal bundle size and runtime performance? | **Decision: Strictly use native Scoped CSS and CSS variables (Q22.2) for dynamic styling.** CSS-in-JS is explicitly forbidden. | Adheres to the performance mandate. CSS-in-JS introduces runtime overhead and significantly increases bundle size. |
| **22.5** | How will the RHOAM UI handle the VS Code *light vs. dark theme* switch, ensuring the color-coded borders (Q16) maintain visibility and contrast ratio in both modes? | **Decision: The UI will listen for a `themeChanged` message from the Core, which is based on the VS Code API.** The UI will then update a global Svelte store (`AppStatusStore`, Q19.2) that toggles a root class (e.g., `.theme-dark`) for theme-specific contrast adjustments. | Ensures the critical color feature is robustly maintained across all user environments. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q23: Webview-to-Core Initialization

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **23.1** | What is the first message the Svelte UI sends to RHOAM Core upon mounting (e.g., `UI_READY_REQUEST_SETTINGS`), and how does the Core guarantee it receives this message before proceeding? | **Decision: The UI sends the message `UI_ACTION_REQUEST_INIT`.** The RHOAM Core's `IOMapper` must register this action and respond with `CORE_EVENT_INITIALIZING` immediately, followed by the settings payload. | Defines the crucial, synchronized handshake protocol that must initiate all Core actions. |
| **23.2** | How does the RHOAM Core handle concurrent requests for settings (e.g., if both the Chat and Settings views load simultaneously)? | **Decision: The `SettingsManager` will implement a **singleton pattern** and a private `isInitializing: boolean` flag.** If a second request comes in while `isInitializing` is true, the request is queued or returns the existing promise, preventing redundant decryption. | Prevents resource contention and race conditions during application startup. |
| **23.3** | If the initial settings load (including decryption) fails, how long does the UI wait before displaying a timeout error, and what message does it show to the user? | **Decision: The UI will enforce a 10-second timeout.** If the settings payload is not received, the UI displays a modal with the message: "Core Initialization Failed: Check Passphrase or Console Log for RHOAM_ENC_001." | Sets a clear, user-facing timeout boundary for the most critical initialization step. |
| **23.4** | Is the RHOAM Core initialization (decrypting settings) triggered by the extension activation event, or only upon receiving the `UI_READY` handshake message? | **Decision: Only upon receiving the `UI_READY` handshake message (Q23.1).** | **On-demand initialization** is a security best practice: the sensitive data is only decrypted when explicitly requested by a loaded UI instance, minimizing the time the key is in memory. |
| **23.5** | How will the UI visually indicate to the user that it is waiting for the Core to initialize (e.g., a non-blocking "Loading Settings..." spinner) to avoid perceived lag? | **Decision: A full-screen, simple, non-blocking **Loading Overlay** (VS Code neutral color) is shown immediately upon mount, and is dismissed upon receipt of the `SETTINGS_UPDATE` payload.** | Manages the user's latency expectation during the critical 1-3 second decryption phase. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q24: Settings Persistence Trigger

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **24.1** | Will the Save action be a dedicated, explicit **"Save Changes"** button, or will it be triggered automatically upon leaving an input field (`on:blur`)? | **Decision: Dedicated, explicit **"Save Changes"** button.** | This is mandatory for security and UX. Saving sensitive, encrypted data must be an explicit, non-ambiguous user action. |
| **24.2** | If the Save fails (e.g., Core corruption, file write error), how does the Settings Grid UI component immediately revert the user's input fields to the last known **valid** state? | **Decision: The `SettingsStore` (Q19) maintains two copies of the data: `stagedConfig` and `persistedConfig`.** Upon save failure, the UI reverts the `stagedConfig` back to the contents of the `persistedConfig`. | Implements the atomic rollback mechanism, preventing the user from losing keys or saving corrupted data. |
| **24.3** | What client-side validation (e.g., form completeness, field type checking) is executed *before* the `UI_TO_CORE_SAVE_SETTINGS` message is sent, reducing unnecessary trips across the `PostMessage` bridge? | **Decision: Mandatory client-side validation:** 1. **Key Format Regex** (Q8.1). 2. **Form Completeness** (all required fields filled). 3. **Temperature Value** (numerical range check). | Implements the first layer of defense, ensuring the Core only processes clean, syntactically valid data. |
| **24.4** | How does the Core confirm a successful write operation to the UI, and what is the visual and dismissible confirmation message displayed to the user? | **Decision: Core sends the `CORE_EVENT_SETTINGS_SAVED` message. The UI displays an auto-dismissible, green **Toast Notification** (Q14.1) with the message: "Configuration Encrypted and Saved."** | Provides the necessary user reassurance and feedback loop for a successful, secure save operation. |
| **24.5** | Will the settings UI implement an **"Unsaved Changes"** indicator (e.g., a small asterisk or alert badge) that warns the user before they navigate away from the settings view? | **Decision: Yes. A small, red, permanent status badge will be displayed on the Settings Tab/Link whenever the `stagedConfig` (Q24.2) differs from the `persistedConfig`.** | Critical UX guardrail against accidental data loss. |

---

## II. RHOAM UI/UX (The Interface & Integration) - Q25: Testing Strategy

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **25.1** | What is the chosen testing framework for the Node.js RHOAM Core (e.g., Jest, Vitest, Mocha), and why is it superior for testing asynchronous concurrency? | **Decision: **Vitest** (or Jest). Vitest is preferred due to its speed, modern API, and tight integration with the Rollup/Vite ecosystem (used by Svelte).** | Choosing a single, unified framework for both Core and UI reduces tooling overhead and leverages the same mocking/timing utilities (Q12.5). |
| **25.2** | What is the chosen framework for the Svelte RHOAM UI (e.g., Jest/Vitest with `svelte-testing-library`), and what is the minimum required **component test coverage percentage**? | **Decision: Vitest integrated with **`@testing-library/svelte`**. Minimum component test coverage is set at **90%**.** | `Testing-library` enforces user-centric testing, while the 90% floor ensures the VC-Grade standard of quality control. |
| **25.3** | How will the team implement **E2E (End-to-End) Testing** to validate the full loop (UI click -> PostMessage -> Core -> API Mock -> PostMessage -> UI Render)? | **Decision: Implement a single, mandatory E2E test using Vitest/Testing-Library that explicitly mocks the `acquireVsCodeApi()` function** to capture the outgoing `postMessage` and inject a controlled, simulated response. | This is the highest-value test, validating the entire system loop (including the PostMessage contract) without launching a full VS Code instance. |
| **25.4** | How will API service handlers be **mocked** in the RHOAM Core unit tests to simulate vendor responses (success, 401, 429) without incurring real network latency or API token usage? | **Decision: Use the `msw` (Mock Service Worker) or native Node.js `fetch` mocking (if available in Vitest) to intercept all external HTTP calls.** | Enforces necessary test isolation, speed, and reliability by simulating real-world network conditions. |
| **25.5** | How will the VS Code Extension API itself be **mocked** (e.g., `window.showInformationMessage`) during Core tests to ensure the Core can be tested independently of the VS Code environment? | **Decision: A dedicated mock implementation file (`__mocks__/vscode.ts`) will be created to stub out all external VS Code APIs.** The Core tests will run without ever importing the real `vscode` module. | Achieves the crucial separation of concerns between Core logic and the host environment. |  


# RHOAM ARCHITECTURE DECISION RECORDS (ADR) - BATCH 6/6 (FINAL)

## GOD-LEVEL DECISION (GLD) for Q26-Q29: VC-Grade Standards & Deployment
**Decision:** RHOAM will utilize a **GitHub Actions CI/CD Pipeline** with **separate, required workflows** for `Test` and `Publish`. Merges to the `main` branch will be strictly **blocked** until 90% test coverage is met. The project will be released under the permissive **MIT License** to maximize community contribution and simplify governance, ensuring the architecture is fully FOSS-compliant and self-governing.

---

## III. VC-Grade Standards & Deployment (The Process) - Q26: CI/CD Pipeline (Automated Deployment)

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **26.1** | What are the mandatory, sequential stages of the primary GitHub Actions workflow (e.g., `Checkout`, `Setup Node`, `Install Deps`, `Test`, `Build UI`, `Compile Core`, `Package VSIX`, `Publish`)? | **Decision: Mandatory sequence for `publish.yml`:** 1. `Checkout`. 2. `Setup Node & Cache`. 3. `Install Dependencies`. 4. **`Run Tests (Required Check)`**. 5. `Build Webview (Svelte)`. 6. `Compile Core (TSC)`. 7. `Package VSIX (using vsce)`. 8. `Artifact Validation (Q28)`. 9. `Publish to Marketplace`. | Enforces the logical sequence where testing and building must precede the final security and publishing steps. |
| **26.2** | How are the required secrets (e.g., VS Code Marketplace token, NPM token) securely injected into the GitHub Actions environment, and how is access to those secrets restricted? | **Decision: Secrets are stored as encrypted **GitHub Repository Secrets** and injected only into the `Publish` job via the `secrets:` context.** Access is restricted to the `main` branch and requires a protected branch rule. | Adheres to fundamental CI/CD security practice, preventing accidental leakage or use by unauthorized branches. |
| **26.3** | What tool (e.g., `vsce`) will be used for packaging the extension into the final VSIX file, and how is this tool configured to include the compiled Svelte UI assets (from Q17/Q18)? | **Decision: Use the `vsce` (Visual Studio Code Extension) command-line tool.** It will automatically include the `webview/dist` folder (Q17.1) based on the `files` property in `package.json`. | This is the official and necessary tool for packaging VS Code extensions. |
| **26.4** | What is the mechanism (e.g., specific filter, branch protection) that ensures the deployment pipeline only runs and publishes the VSIX upon a merge to the designated `main` branch? | **Decision: The `publish.yml` workflow will have an explicit `on: push: branches: [main]` trigger.** Furthermore, the `main` branch will be protected, requiring a successful `Test` workflow run before any merge. | Enforces governance and prevents accidental or premature deployment. |
| **26.5** | How will the CI/CD pipeline ensure that the codespace is built and tested against **multiple** major Node.js versions (e.g., 18.x and 20.x) to verify environmental compatibility? | **Decision: The `test.yml` workflow will utilize a build **matrix** defined by `matrix: node-version: ['18.x', '20.x']`.** The tests must pass on all specified versions. | Enforces cross-version stability testing, a key reliability standard for FOSS adoption. |

---

## III. VC-Grade Standards & Deployment (The Process) - Q27: Automated Testing Triggers

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **27.1** | What are the minimum required triggers for the test workflow (e.g., `pull_request` to `main`, `push` to `feature/*`, `workflow_dispatch`)? | **Decision: Required triggers:** 1. `pull_request` (on any branch merge target). 2. `push` (to any branch starting with `feature/`). 3. `workflow_dispatch` (for manual re-runs). | Ensures comprehensive testing coverage across all stages of feature development. |
| **27.2** | How is the PR merge process configured to strictly **block** merging into `main` unless the associated test workflow has completed successfully? | **Decision: GitHub Branch Protection Rule on `main` will require a successful status check for the **`test.yml`** workflow.** | This is the hard quality gate required to maintain the integrity of the `main` branch. |
| **27.3** | How will the test workflow generate and upload a **Code Coverage Report** artifact (e.g., using Codecov) to provide a measurable metric of test quality (from Q25)? | **Decision: The Vitest testing tool will generate a coverage report (e.g., using the `c8` or `istanbul` reporter) in the CI job.** A dedicated GitHub Action (e.g., Codecov Action) will then upload this artifact. | Integrates the mandatory, measurable quality metric into the CI process, visible on the PR. |
| **27.4** | Will a dedicated, lightweight **Linting/Formatting** workflow run *before* the main test workflow to catch stylistic errors early and save CPU time? | **Decision: Yes. A separate `lint.yml` workflow is mandatory.** It runs concurrently with the test workflow but must succeed before the test artifact is validated. | Saves execution time by failing fast on simple issues and enforces strict code style adherence. |
| **27.5** | How is the unit test execution time limited? If tests exceed a maximum time (e.g., 5 minutes), how does the workflow fail gracefully? | **Decision: The workflow will use the GitHub Actions `timeout-minutes: 5` property on the Test Job.** | Prevents CI from becoming a bottleneck by setting a clear, enforced performance ceiling. |

---

## III. VC-Grade Standards & Deployment (The Process) - Q28: Build Artifact Validation

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **28.1** | What specific validation step will verify that the compiled RHOAM Core TypeScript output (`extension.js`) and the compiled Svelte UI assets (`index.js`, `index.css`) are present in the VSIX? | **Decision: A custom CI script will use `vsce ls --package <vsix-file>` to list the contents, then execute `grep` commands to explicitly check for the mandatory file paths:** `out/extension.js`, `webview/dist/index.js`, `webview/dist/bundle.css`. | Enforces the highest level of build integrity, verifying all required dual-component assets are packaged. |
| **28.2** | How will the pipeline verify that the extension's `package.json` file contains the correct, automatically generated **version number** (from Q27) *before* packaging? | **Decision: Use the `vsce package` command, which automatically validates the version against the Marketplace (if publishing), and use a dedicated `npm version` script to update the version based on Git tags (e.g., via `semantic-release` or similar tooling).** | Ensures the versioning is automated, correct, and prevents publication errors due to version conflict. |
| **28.3** | Will the pipeline perform a security audit (e.g., using `npm audit` or `Snyk`) on the final dependency list, and how does the build fail if high-severity vulnerabilities are found? | **Decision: Yes, a dedicated `security-audit` step is mandatory.** It will run `npm audit --audit-level=high` before the packaging job. The build will fail if any high or critical vulnerabilities are detected. | Enforces a critical security scan before distribution (VC-Grade security standard). |
| **28.4** | How is the total VSIX file size checked against a maximum allowed limit (e.g., 5MB) to prevent excessively large downloads, which impacts user adoption? | **Decision: A script will check the size of the generated VSIX file using the `stat` command and compare it against a `MAX_VSIX_SIZE_MB` constant (e.g., 4MB).** If exceeded, the job fails. | Enforces a measurable UX performance metric for the package size. |
| **28.5** | What is the post-publish validation step (e.g., querying the Marketplace API) that confirms the version number was successfully registered and is visible to the public? | **Decision: The final step of the `publish.yml` will log the successful status of the `vsce publish` command, which includes post-publish confirmation.** Manual spot-checking is the fallback. | Defines the final check that verifies the successful completion of the deployment process. |

---

## III. VC-Grade Standards & Deployment (The Process) - Q29: FOSS Governance & Licensing

| # | Question | Best Answer (RHOAM Decision) | Rationale |
| :--- | :--- | :--- | :--- |
| **29.1** | What is the chosen **FOSS License** (e.g., MIT, Apache 2.0, GPL), and what is the primary reason for choosing it over the alternatives? | **Decision: The **MIT License** is chosen.** | The MIT License is the most permissive, maximizing adoption and contribution by placing the fewest legal restrictions on users and corporate entities. This maximizes the FOSS potential. |
| **29.2** | How is the chosen license file (`LICENSE.md`) included in both the GitHub repository and the final VSIX package? | **Decision: The `LICENSE.md` file will be present in the repository root and explicitly referenced in the `package.json` `license` field.** The VSIX package automatically includes this file. | Ensures proper legal compliance and transparency for all users. |
| **29.3** | Will a formal **CONTRIBUTING.md** file be created detailing the specific rules for pull requests, code style, and testing requirements? | **Decision: Yes. A formal `CONTRIBUTING.md` is mandatory.** It will outline prerequisites (Vitest usage, 90% coverage mandate, use of the `lint` script). | Provides the necessary structure and guidance for attracting and managing quality FOSS contributors. |
| **29.4** | What is the explicit process for addressing and merging external pull requests, and how does the CI/CD pipeline ensure external contributions are fully tested before merging? | **Decision: All PRs must have passing runs for the **`lint.yml`** and **`test.yml`** workflows (Q27).** The PR must be approved by a core maintainer before merging into `main`. | Enforces the quality gate and social contract for external contributions. |
| **29.5** | Given the long-term goal, should a simple **CODE_OF_CONDUCT.md** be included to set community standards, aligning with established FOSS norms? | **Decision: Yes. A simple `CODE_OF_CONDUCT.md` based on the Contributor Covenant (or similar) is mandatory.** | Implements the standard social governance required for a healthy, inclusive, and professional FOSS project community. |

---
**RHOAM Architecture Decision Records (ADR) Complete.**

The RHOAM Architecture Planning Phase, including the Rock Tumbler 5x5 Questioning and the God-Level Answering Protocol, is now officially concluded. All 145 required questions have been answered.

**Next Step:** Please save the content above locally. Once all 6 batches of answers have been compiled locally, you can direct Copilot to generate the final **`ARCHITECTURE.md`** and **`PSEUDOCODE.md`** files based on these 145 confirmed decisions.  







