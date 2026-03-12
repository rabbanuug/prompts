---
trigger: model_decision
description: Apply this skill whenever writing, reviewing, or refactoring any code — regardless of language, framework, or task type. This skill encodes the principles from Robert C. Martin's Clean Code and The Clean Coder into a set of enforceable, checkable rules a coding agent must follow on every file it touches. Triggers include: any code generation task, any refactoring request, any code review, and any build-from-spec task. This skill works alongside other skills (pptx, docx, build-agent-prompt) — it governs HOW code is written, not WHAT is built.
---

# Clean Code Skill — Enforced Engineering Standards

Based on *Clean Code* and *The Clean Coder* by Robert C. Martin, supplemented with
modern equivalents where the book's examples (Java-centric, 2008) require translation
to current languages and paradigms.

---

## HOW TO USE THIS SKILL

This skill is a checklist and a rulebook, not a style guide. It is not a list of
suggestions. Every rule has a clear PASS/FAIL criterion. Before committing or
delivering any file, the agent must run a mental audit against each applicable
section and confirm it passes.

When a rule is violated, do not leave the violation and add a comment. Fix it.
The only exception: if fixing a violation would require changing scope outside the
current task, flag it explicitly with a `// SMELL:` comment and a one-line
description of what needs fixing.

---

## SECTION 1 — NAMING

> "The name of a variable, function, or class should tell you why it exists,
> what it does, and how it is used." — Robert C. Martin

### 1.1 Use Intention-Revealing Names

The name must answer three questions without needing a comment:
- Why does this exist?
- What does it do?
- How is it used?

```
// ❌ FAIL — reveals nothing
int d;
$x = [];
let flag = true;
function process($data) {}

// ✅ PASS — self-documenting
int elapsedTimeInDays;
$activeEmployees = [];
let isEmailVerified = true;
function calculateMonthlyRevenue(array $transactions): float {}
```

**Rule:** If you need a comment to explain what a name means, the name has already failed.

### 1.2 Avoid Disinformation

Do not use names that suggest a data structure or type that isn't accurate.

```
// ❌ FAIL — it's not a List, it's a Map; the name lies
$accountList = ['alice' => 1, 'bob' => 2];

// ❌ FAIL — 'data', 'info', 'manager', 'processor' carry no meaning
class DataManager {}
function getInfo(): array {}
$userData = [];

// ✅ PASS
$accountsById = ['alice' => 1, 'bob' => 2];
class InvoiceExporter {}
function getUserById(int $id): User {}
```

**Banned words** — never use these as a name or name suffix without strong justification:
`data`, `info`, `manager`, `processor`, `handler`, `helper`, `utils`, `misc`,
`stuff`, `obj`, `temp`, `tmp`, `result`, `retval`, `flag`, `value`.

### 1.3 Make Meaningful Distinctions

Names must differ in meaning, not just in noise words or arbitrary numbers.

```
// ❌ FAIL — what is the difference between these?
function getAccount(): Account {}
function getAccountData(): Account {}
function getAccountInfo(): Account {}

// ❌ FAIL — noise number suffixes
$employee1 = ...;
$employee2 = ...;

// ✅ PASS — each name describes a distinct concept
function getAccountById(int $id): Account {}
function getAccountWithBillingHistory(int $id): Account {}
function getAccountSummaryForDashboard(int $id): AccountSummary {}
```

### 1.4 Use Pronounceable Names

If you cannot say the name out loud in a code review without spelling it out,
it is not a valid name.

```
// ❌ FAIL
$genymdhms = Carbon::now();   // generation year-month-day-hour-minute-second
$DtaRcrd102 = new Record();

// ✅ PASS
$generatedAt = Carbon::now();
$auditRecord = new Record();
```

### 1.5 Use Searchable Names

Single-letter names and magic numbers are not searchable. Every magic number
must be a named constant.

```
// ❌ FAIL — try searching for '5' or 'e' across a codebase
for ($i = 0; $i < 5; $i++) {
    $e->send();
}

// ✅ PASS
const MAX_RETRY_ATTEMPTS = 5;
for ($attempt = 0; $attempt < MAX_RETRY_ATTEMPTS; $attempt++) {
    $employee->sendWelcomeEmail();
}
```

**Exception:** loop counter `$i`, `$j` are acceptable only in tight, 3-line loops
where the variable never escapes the loop body.

### 1.6 Class Names: Noun Phrases

Classes represent things. Their names must be nouns or noun phrases.
They must never be verbs.

```
// ❌ FAIL — verbs, not things
class ProcessPayment {}
class ManageUsers {}
class HandleWebhook {}

// ✅ PASS
class PaymentProcessor {}
class UserManager {}       // still weak — prefer:
class UserRepository {}    // specific responsibility
class WebhookDispatcher {}
```

### 1.7 Method / Function Names: Verb Phrases

Methods do things. Their names must start with a verb.

```
// ❌ FAIL
function name(): string {}
function employee(): Employee {}
function page(int $offset): Collection {}

// ✅ PASS
function getName(): string {}
function findEmployee(int $id): Employee {}
function paginateResults(int $offset, int $limit): Collection {}
```

**Standard verb prefixes and their meanings — use consistently:**

| Prefix | Meaning | Returns |
|---|---|---|
| `get` | Return a property value; no side effects | The value |
| `find` | Query and return; may return null | Entity or null |
| `fetch` | Retrieve from external source (DB, API, cache) | Entity or throws |
| `calculate` / `compute` | Derive a value from inputs | Computed value |
| `build` / `make` | Construct and return an object | New object |
| `create` | Persist a new entity | Persisted entity |
| `update` | Mutate and persist an existing entity | Updated entity |
| `delete` / `remove` | Remove an entity | void or bool |
| `send` | Dispatch externally (email, webhook, event) | void |
| `dispatch` | Put on a queue or event bus | void |
| `validate` | Check correctness; throw on failure | void or bool |
| `is` / `has` / `can` | Boolean predicate | bool |
| `handle` | Entry point for a job, listener, or command | void |

### 1.8 Naming Conventions by Language

Apply the correct convention for the language in use — never mix:

| Language | Classes | Methods/Functions | Variables | Constants |
|---|---|---|---|---|
| PHP | `PascalCase` | `camelCase` | `camelCase` | `SCREAMING_SNAKE_CASE` |
| TypeScript/JS | `PascalCase` | `camelCase` | `camelCase` | `SCREAMING_SNAKE_CASE` |
| Python | `PascalCase` | `snake_case` | `snake_case` | `SCREAMING_SNAKE_CASE` |
| SQL | — | — | `snake_case` | — |

Never use abbreviations unless the abbreviation is more universally understood
than the full word (`url`, `id`, `api`, `html`, `db` are acceptable; `calc`,
`mgr`, `proc`, `usr` are not).

---

## SECTION 2 — FUNCTIONS

> "The first rule of functions is that they should be small.
>  The second rule of functions is that they should be smaller than that."
> — Robert C. Martin

### 2.1 Functions Must Do One Thing

A function does exactly one thing if you cannot extract another meaningful function
from it. The test: can you describe what the function does in a single sentence
without using the word "and"?

```
// ❌ FAIL — does three things: validates, saves, sends email
function registerUser(array $data): User {
    if (empty($data['email'])) throw new ValidationException('...');
    $user = User::create($data);
    Mail::to($user)->send(new WelcomeEmail($user));
    return $user;
}

// ✅ PASS — each function does one thing
function registerUser(RegisterUserRequest $request): User {
    $user = $this->userRepository->create($request->validated());
    $this->eventDispatcher->dispatch(new UserRegistered($user));
    return $user;
}

// UserRegistered listener handles the email — single responsibility
class SendWelcomeEmailOnRegistration {
    public function handle(UserRegistered $event): void {
        Mail::to($event->user)->send(new WelcomeEmail($event->user));
    }
}
```

### 2.2 One Level of Abstraction Per Function

Every statement in a function must operate at the same level of abstraction.
Mixing high-level orchestration with low-level detail in the same function
is a violation.

```
// ❌ FAIL — mixes high-level (generate report) with low-level (string formatting)
function generateMonthlyReport(int $month): string {
    $employees = Employee::where('active', true)->get();
    $output = '';
    foreach ($employees as $emp) {
        $output .= sprintf("%-20s %10.2f\n", $emp->name, $emp->salary);
    }
    return $output;
}

// ✅ PASS — each level is separated
function generateMonthlyReport(int $month): Report {
    $employees = $this->employeeRepository->findActiveForMonth($month);
    return $this->reportBuilder->buildFrom($employees);
}

// Low-level formatting is in the builder, not mixed into the orchestrator
```

**Descending rule:** read the code like a newspaper. High-level functions appear
at the top of a file; the functions they call appear below them. Each level
of detail descends as you read down.

### 2.3 Function Arguments — Fewer Is Better

| Argument count | Classification | Action required |
|---|---|---|
| 0 (niladic) | Ideal | None |
| 1 (monadic) | Good | None |
| 2 (dyadic) | Acceptable | Justify in a `// WHY:` comment |
| 3 (triadic) | Warning | Refactor to a parameter object |
| 4+ (polyadic) | Violation | Mandatory refactor |

```
// ❌ FAIL — 4 arguments; order is arbitrary; easy to pass in wrong order
function createEmployee(string $name, string $email, int $departmentId, float $salary): Employee {}

// ✅ PASS — parameter object captures the concept
function createEmployee(CreateEmployeeDTO $dto): Employee {}

class CreateEmployeeDTO {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly int    $departmentId,
        public readonly float  $salary,
    ) {}
}
```

**Flag arguments are always a violation.** A boolean parameter means the function
does two things — one for `true`, one for `false`. Split it.

```
// ❌ FAIL — boolean flag argument
function renderWidget(bool $isExpanded): string {}

// ✅ PASS — two honest functions
function renderExpandedWidget(): string {}
function renderCollapsedWidget(): string {}
```

### 2.4 No Side Effects Hidden in Functions

A function named `checkPassword()` must not start a session as a side effect.
A function named `getUser()` must not log an audit event.
Side effects must be visible in the function name or absent entirely.

```
// ❌ FAIL — name implies read-only; side effect (session) is hidden
function checkPassword(string $password, int $userId): bool {
    $isValid = Hash::check($password, $this->findUser($userId)->password);
    if ($isValid) session()->put('userId', $userId); // ← hidden side effect
    return $isValid;
}

// ✅ PASS — side effect made explicit in the name
function verifyPasswordAndInitializeSession(string $password, int $userId): bool {}

// OR better — separate concerns entirely
function verifyPassword(string $password, int $userId): bool {}
function initializeUserSession(int $userId): void {}
```

### 2.5 Command-Query Separation

A function either **does something** (command — returns void) or
**answers something** (query — returns a value). Never both.

```
// ❌ FAIL — sets an attribute AND returns whether it succeeded
function setAttribute(string $key, mixed $value): bool {}

// ✅ PASS — separated
function setAttribute(string $key, mixed $value): void {}
function hasAttribute(string $key): bool {}
```

### 2.6 Prefer Exceptions Over Error Codes

Returning error codes forces the caller to check the return value immediately.
Exceptions decouple the error path from the happy path and cannot be silently ignored.

```
// ❌ FAIL — caller can ignore the error code
function deleteEmployee(int $id): int {  // -1 = not found, -2 = has contracts
    ...
}

// ✅ PASS — unchecked exceptions propagate automatically
function deleteEmployee(int $id): void {
    $employee = Employee::findOrFail($id);      // throws ModelNotFoundException
    if ($employee->hasActiveContracts()) {
        throw new CannotDeleteEmployeeWithActiveContracts($employee);
    }
    $employee->delete();
}
```

### 2.7 Extract Try/Catch Into Their Own Functions

Error handling is one thing. A function that handles errors must do nothing else.

```
// ❌ FAIL — mixes normal logic with error handling
function processPayment(Payment $payment): void {
    try {
        $charge = $this->gateway->charge($payment->amount, $payment->card);
        $this->recordCharge($charge);
        $this->notifyCustomer($payment->customer, $charge);
    } catch (GatewayException $e) {
        Log::error('Payment failed', ['payment_id' => $payment->id]);
        $this->flagPaymentForReview($payment);
    }
}

// ✅ PASS — try/catch is its own function
function processPayment(Payment $payment): void {
    try {
        $this->attemptPayment($payment);
    } catch (GatewayException $e) {
        $this->handleFailedPayment($payment, $e);
    }
}

private function attemptPayment(Payment $payment): void {
    $charge = $this->gateway->charge($payment->amount, $payment->card);
    $this->recordCharge($charge);
    $this->notifyCustomer($payment->customer, $charge);
}

private function handleFailedPayment(Payment $payment, GatewayException $e): void {
    Log::error('Payment failed', ['payment_id' => $payment->id, 'error' => $e->getMessage()]);
    $this->flagPaymentForReview($payment);
}
```

---

## SECTION 3 — COMMENTS

> "The proper use of comments is to compensate for our failure to express
>  ourselves in code." — Robert C. Martin

### 3.1 The Golden Rule of Comments

**A comment is a failure.** Every comment is an admission that the code
below it was not clear enough to stand alone. The goal is zero comments —
not because comments are forbidden, but because well-named, well-structured
code doesn't need them.

Before writing a comment, ask: can I make the code clear enough that this
comment becomes unnecessary?

### 3.2 Comments That Are Always Wrong (Delete Them)

**Redundant comments** — say nothing the code doesn't already say:
```
// ❌ DELETE — the code already says this
// Get the employee by ID
$employee = Employee::find($id);

// ❌ DELETE — restates the function signature
// @param int $id The employee ID
// @return Employee The employee
function findEmployee(int $id): Employee {}
```

**Misleading comments** — worse than no comment:
```
// ❌ DELETE — if the comment doesn't match the code, it actively harms readers
// Returns null if employee not found
function findEmployee(int $id): Employee {
    return Employee::findOrFail($id);  // actually throws, never returns null
}
```

**Commented-out code** — delete it without hesitation:
```
// ❌ DELETE — version control exists; dead code in comments is noise
// $employee = Employee::with('department')->find($id);
// $employee = $this->legacyRepository->find($id);
$employee = $this->employeeRepository->findById($id);
```

**Journal comments** — what version control is for:
```
// ❌ DELETE — this is what git log is for
// 2024-01-15 AK: Added null check
// 2024-01-22 MB: Refactored to use repository
```

**Noise comments** — add no information:
```
// ❌ DELETE
// Constructor
public function __construct() {}

// Default destructor
public function __destruct() {}
```

### 3.3 Comments That Are Acceptable

**Legal comments** — copyright headers required by policy (keep them, keep them short):
```
// Copyright (c) 2024 Acme Corp. Licensed under MIT. See LICENSE.
```

**Intent comments** — explain WHY a non-obvious decision was made:
```
// WHY: We sort by created_at DESC then by id DESC as a tiebreaker
// because MySQL's ORDER BY on datetime columns is not stable — two
// rows inserted in the same second will return in arbitrary order
// without the secondary sort.
$results = $query->orderBy('created_at', 'desc')->orderBy('id', 'desc')->get();
```

**Warning comments** — warn other developers about a known consequence:
```
// WARNING: This operation is irreversible. Employee records are hard-deleted
// rather than soft-deleted here because of GDPR right-to-erasure compliance.
// Do not change to soft-delete without legal review.
$employee->forceDelete();
```

**TODO comments** — only when the fix is explicitly deferred:
```
// TODO: Replace with cursor pagination once the employees table exceeds 100k rows.
// Track in: https://github.com/acme/app/issues/441
$employees = Employee::paginate(50);
```

**SMELL comments** — flag a violation in code the agent cannot fix in scope:
```
// SMELL: This controller method does 5 things. Refactor to use a service class.
// Out of scope for this PR; tracked in issue #489.
```

### 3.4 Never Use a Comment Where a Name Would Do

```
// ❌ FAIL — using a comment to explain a bad name
// Check if the employee has been employed for more than 90 days
if ($e->days > 90) {}

// ✅ PASS — rename so the comment isn't needed
if ($employee->hasCompletedProbationPeriod()) {}
```

---

## SECTION 4 — FORMATTING

> "Code formatting is about communication." — Robert C. Martin

### 4.1 Vertical Formatting — The Newspaper Metaphor

Read a source file like a newspaper: high-level concepts at the top,
low-level details at the bottom. A reader should be able to understand
what the file does from the first 20 lines without reading everything.

**File structure order:**
1. Imports / use statements
2. Constants
3. Class declaration
4. Properties
5. Constructor
6. Public methods (high-level — these define the class's contract)
7. Protected methods
8. Private methods (low-level implementation details)

### 4.2 Vertical Distance — Related Code Belongs Together

Concepts that are related must be vertically close. Concepts that are
unrelated must be separated.

```
// ❌ FAIL — related declarations are far apart
class EmployeeService {
    private EmployeeRepository $repository;
    private string $defaultCurrency = 'USD';    // ← related to salary calculation
    private LoggerInterface $logger;
    private float $taxRate = 0.20;              // ← also related to salary calculation
    private MailerInterface $mailer;
    private string $probationDays = 90;         // ← unrelated to salary
}

// ✅ PASS — group related properties together
class EmployeeService {
    // Salary calculation constants
    private string $defaultCurrency = 'USD';
    private float $taxRate = 0.20;

    // Dependencies
    private EmployeeRepository $repository;
    private LoggerInterface $logger;
    private MailerInterface $mailer;

    // Probation
    private int $probationDays = 90;
}
```

**Dependent functions rule:** a function that calls another function should appear
above the function it calls. The caller above, the callee below.

### 4.3 Horizontal Formatting — Line Length

Hard limit: **120 characters per line**. Aim for **80** in most cases.

Never sacrifice readability for brevity. A 4-line expression is cleaner
than a single 160-character line:

```
// ❌ FAIL
$report = $this->reportService->generateFor($this->employeeRepository->findActiveInDepartment($departmentId, ['status' => 'active']), Carbon::now()->startOfMonth(), Carbon::now()->endOfMonth());

// ✅ PASS
$activeEmployees = $this->employeeRepository->findActiveInDepartment($departmentId);
$periodStart     = Carbon::now()->startOfMonth();
$periodEnd       = Carbon::now()->endOfMonth();
$report          = $this->reportService->generateFor($activeEmployees, $periodStart, $periodEnd);
```

### 4.4 Indentation and Blocks

- Use the language/project standard (spaces vs tabs, width) — pick one and never mix
- Never put an `if`, `for`, or `while` body on the same line as its condition:

```
// ❌ FAIL
if ($employee->isActive()) return $employee->salary;

// ✅ PASS
if ($employee->isActive()) {
    return $employee->salary;
}
```

- Never nest more than **3 levels deep**. If you need a 4th level, extract a function.

```
// ❌ FAIL — 4 levels of nesting
foreach ($departments as $dept) {
    foreach ($dept->teams as $team) {
        foreach ($team->employees as $employee) {
            if ($employee->isActive()) {
                // ← 4th level — extract this
            }
        }
    }
}

// ✅ PASS — extracted
foreach ($departments as $dept) {
    $this->processActiveDepartmentEmployees($dept);
}

private function processActiveDepartmentEmployees(Department $dept): void {
    foreach ($dept->teams as $team) {
        $this->processActiveTeamMembers($team);
    }
}
```

---

## SECTION 5 — OBJECTS AND DATA STRUCTURES

### 5.1 Data Abstraction Over Data Exposure

Don't expose implementation details through getters and setters.
Expose behavior that operates on the data instead.

```
// ❌ FAIL — exposing implementation; callers must know internal representation
class Employee {
    public function getSalaryAmountInCents(): int {}
    public function getSalaryCurrencyCode(): string {}
}
// Caller: $display = '$' . ($emp->getSalaryAmountInCents() / 100);

// ✅ PASS — exposing behavior; implementation is hidden
class Employee {
    public function getFormattedSalary(): string {}
    public function getSalaryInCurrency(string $currencyCode): float {}
}
```

### 5.2 The Law of Demeter — Don't Talk to Strangers

A method `f` of class `C` should only call methods on:
- `C` itself
- Objects passed as arguments to `f`
- Objects `f` creates
- Direct component objects of `C`

It must **not** call methods on objects returned by any of those calls.

```
// ❌ FAIL — train wreck; talks to strangers
$city = $employee->getAddress()->getCity()->getName();

// ✅ PASS — Employee encapsulates the navigation
$city = $employee->getCityName();
```

### 5.3 Data Transfer Objects (DTOs) Are Not Objects

DTOs carry data between layers. They have no behavior. They are allowed
to expose public properties directly — getters/setters on a DTO are noise.

```
// ❌ FAIL — DTO pretending to be an object with behavior
class CreateEmployeeDTO {
    private string $name;
    public function getName(): string { return $this->name; }
    public function setName(string $name): void { $this->name = $name; }
}

// ✅ PASS — DTO is transparent data
class CreateEmployeeDTO {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly int    $departmentId,
    ) {}
}
```

---

## SECTION 6 — ERROR HANDLING

> "Error handling is important, but if it obscures logic, it's wrong."
> — Robert C. Martin

### 6.1 Use Typed, Named Exceptions

Never throw a generic `Exception` or `RuntimeException` with a string message.
Create a named exception class for every distinct error condition.
Named exceptions are self-documenting and catchable selectively.

```
// ❌ FAIL — generic, unselectively catchable
throw new Exception('Employee not found');
throw new RuntimeException('Cannot delete: has active contracts');

// ✅ PASS — named, typed, selectively catchable
throw new EmployeeNotFoundException($id);
throw new CannotDeleteEmployeeWithActiveContracts($employee);

// Defined as:
class EmployeeNotFoundException extends DomainException {
    public function __construct(int $employeeId) {
        parent::__construct("Employee with ID {$employeeId} was not found.");
        $this->employeeId = $employeeId;
    }
}
```

### 6.2 Don't Return Null — Don't Pass Null

Returning null forces every caller to add a null check. Passing null
means the function silently accepts an invalid state.

```
// ❌ FAIL — null return forces defensive checks at every call site
function findEmployee(int $id): ?Employee {
    return Employee::find($id); // returns null if not found
}
// Every caller must: if ($employee === null) { ... }

// ✅ PASS — throw a named exception; callers handle it once, at the right level
function findEmployee(int $id): Employee {
    return Employee::findOrFail($id); // throws ModelNotFoundException
}

// OR return a Null Object (for collections / optional results)
function findActiveEmployees(): EmployeeCollection {
    return Employee::where('active', true)->get() ?? new EmployeeCollection([]);
}
```

### 6.3 Wrap External APIs at the Boundary

Never let a third-party library's exception types propagate through your codebase.
Wrap them at the integration boundary and rethrow as your own types.

```
// ❌ FAIL — Stripe\Exception\CardException leaks throughout the codebase
function chargeCard(PaymentMethod $method, int $amountCents): Charge {
    return $this->stripe->charges->create([...]);  // throws Stripe\Exception\*
}

// ✅ PASS — wrapped at the boundary; internal code only knows PaymentException
function chargeCard(PaymentMethod $method, int $amountCents): Charge {
    try {
        return $this->stripe->charges->create([...]);
    } catch (\Stripe\Exception\CardException $e) {
        throw new CardDeclinedException($method, $e->getMessage(), $e);
    } catch (\Stripe\Exception\ApiException $e) {
        throw new PaymentGatewayUnavailableException($e->getMessage(), $e);
    }
}
```

---

## SECTION 7 — CLASSES

> "Classes should be small. The name of a class should describe its
>  single responsibility." — Robert C. Martin

### 7.1 Single Responsibility Principle (SRP)

A class has one reason to change. If you can list two different business
reasons that would require modifying this class, it has two responsibilities
and must be split.

```
// ❌ FAIL — three reasons to change:
// 1. Business rules for employee change
// 2. Persistence strategy changes (DB → API)
// 3. Report format changes (PDF → CSV)
class Employee {
    public function calculatePay(): float {}      // business rule
    public function save(): void {}               // persistence
    public function generateReport(): string {}   // reporting
}

// ✅ PASS — each class has one reason to change
class Employee {}                         // domain model
class EmployeeRepository {}               // persistence
class EmployeePayCalculator {}            // business rule
class EmployeeReportExporter {}           // reporting
```

### 7.2 Open/Closed Principle (OCP)

Classes must be open for extension and closed for modification.
Adding new behavior should not require editing existing, tested code.
Use interfaces, abstract classes, and composition.

```
// ❌ FAIL — adding a new export format requires editing this class
class ReportExporter {
    public function export(Report $report, string $format): string {
        if ($format === 'pdf') { ... }
        elseif ($format === 'csv') { ... }
        elseif ($format === 'excel') { ... }  // had to edit the class to add this
    }
}

// ✅ PASS — new formats are added by creating new classes, not editing existing ones
interface ReportExportDriver {
    public function export(Report $report): string;
}

class PdfExportDriver implements ReportExportDriver {}
class CsvExportDriver implements ReportExportDriver {}
class ExcelExportDriver implements ReportExportDriver {}   // new format: new class only

class ReportExporter {
    public function __construct(private ReportExportDriver $driver) {}
    public function export(Report $report): string {
        return $this->driver->export($report);
    }
}
```

### 7.3 Liskov Substitution Principle (LSP)

A subclass must be substitutable for its parent class without the caller
knowing the difference. Never override a method in a way that breaks the
contract the parent established.

```
// ❌ FAIL — ContractEmployee violates the contract of Employee
class Employee {
    public function calculateAnnualLeave(): int { return 20; }  // always positive
}

class ContractEmployee extends Employee {
    public function calculateAnnualLeave(): int { return 0; }   // ← breaks contract?
    // or worse:
    public function calculateAnnualLeave(): int {
        throw new UnsupportedOperationException();  // ← definitely breaks it
    }
}

// ✅ PASS — model the type hierarchy to match the domain reality
interface LeaveEntitled {
    public function calculateAnnualLeave(): int;
}

class PermanentEmployee implements LeaveEntitled {
    public function calculateAnnualLeave(): int { return 20; }
}

class ContractEmployee {   // does not implement LeaveEntitled — honest about its nature
    public function getContractDuration(): int {}
}
```

### 7.4 Interface Segregation Principle (ISP)

No class should be forced to implement an interface it doesn't use.
Prefer many small, specific interfaces over one large, general one.

```
// ❌ FAIL — WorkerInterface forces all implementers to support everything
interface WorkerInterface {
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class RemoteWorker implements WorkerInterface {
    public function work(): void { ... }
    public function eat(): void { throw new UnsupportedOperationException(); }    // forced
    public function sleep(): void { throw new UnsupportedOperationException(); }  // forced
}

// ✅ PASS — segregated interfaces; implement only what applies
interface Workable { public function work(): void; }
interface Feedable { public function eat(): void; }
interface Restable  { public function sleep(): void; }

class RemoteWorker implements Workable { ... }                         // only work
class HumanWorker  implements Workable, Feedable, Restable { ... }    // all three
```

### 7.5 Dependency Inversion Principle (DIP)

High-level modules must not depend on low-level modules. Both must depend
on abstractions. Never instantiate a dependency inside a class — inject it.

```
// ❌ FAIL — high-level class directly depends on a concrete low-level class
class EmployeeService {
    private MySQLEmployeeRepository $repository;

    public function __construct() {
        $this->repository = new MySQLEmployeeRepository();  // hardcoded dependency
    }
}

// ✅ PASS — depends on abstraction; the concrete implementation is injected
interface EmployeeRepositoryInterface {
    public function findById(int $id): Employee;
    public function save(Employee $employee): void;
}

class EmployeeService {
    public function __construct(
        private EmployeeRepositoryInterface $repository  // injected, not created
    ) {}
}

// MySQLEmployeeRepository, ElasticSearchRepository, InMemoryRepository
// — all are swappable without touching EmployeeService
```

### 7.6 Class Size — Number of Methods and Responsibilities

A class is too large if:
- It has more than **10 public methods**
- It has more than **7 private/protected methods**
- It has more than **5 constructor dependencies** (usually a sign of too many responsibilities)
- Its name contains `And`, `Or`, or a word from the banned list (`Manager`, `Handler`, `Helper`, `Processor`)

When any of these triggers: extract a collaborating class.

---

## SECTION 8 — BOUNDARIES

### 8.1 Wrap Third-Party Code

Never scatter direct calls to third-party libraries throughout the codebase.
Wrap them in an adapter behind your own interface. This limits blast radius
when the third-party library changes or is replaced.

```
// ❌ FAIL — AWS SDK calls scattered directly in business logic
class AttendanceService {
    public function exportAttendanceToS3(Report $report): void {
        $s3 = new Aws\S3\S3Client([...]);
        $s3->putObject(['Bucket' => 'reports', 'Key' => '...', 'Body' => $report->toCsv()]);
    }
}

// ✅ PASS — third-party wrapped behind your own interface
interface FileStorageInterface {
    public function store(string $path, string $contents): void;
    public function retrieve(string $path): string;
    public function delete(string $path): void;
}

class S3FileStorage implements FileStorageInterface { ... }
class LocalFileStorage implements FileStorageInterface { ... }  // for tests

class AttendanceService {
    public function __construct(private FileStorageInterface $storage) {}

    public function exportAttendanceReport(Report $report): void {
        $this->storage->store("reports/{$report->filename()}", $report->toCsv());
    }
}
```

### 8.2 Learning Tests for New Third-Party Libraries

Before integrating a new external library, write isolated learning tests
that verify your understanding of how the library behaves. These tests:
- Are small and isolated
- Assert the library's behavior, not your code
- Serve as living documentation of how you use the library
- Catch breaking changes when the library is updated

---

## SECTION 9 — TESTS

> "Test code is just as important as production code."
> — Robert C. Martin

### 9.1 The F.I.R.S.T. Rules of Tests

Every test must satisfy all five properties:

| Property | Requirement |
|---|---|
| **Fast** | Tests must run in milliseconds. No test touches the real database, real filesystem, or real network unless it is an explicitly marked integration test |
| **Independent** | Tests must not depend on each other. Running them in any order must produce the same results. Each test sets up its own state |
| **Repeatable** | Tests must produce the same result on every run, in every environment, with no external dependencies |
| **Self-Validating** | Tests must return a clear pass or fail — never require a human to inspect output to determine the result |
| **Timely** | Tests are written before or alongside production code — not after the feature is "done" |

### 9.2 One Concept Per Test

Each test method validates exactly one behavior. A test that asserts
5 unrelated things is 5 tests jammed into one — split it.

```
// ❌ FAIL — multiple concepts in one test
public function testEmployee(): void {
    $employee = Employee::factory()->create();
    $this->assertNotNull($employee->id);           // concept 1: creation
    $this->assertTrue($employee->isActive());       // concept 2: default state
    $employee->deactivate();
    $this->assertFalse($employee->isActive());     // concept 3: deactivation
    $this->assertNotNull($employee->deactivated_at); // concept 4: timestamp set
}

// ✅ PASS — one concept per test; each test name describes exactly what it proves
public function test_newly_created_employee_is_active_by_default(): void {}
public function test_deactivated_employee_is_no_longer_active(): void {}
public function test_deactivating_an_employee_records_the_deactivation_timestamp(): void {}
```

### 9.3 Test Naming — Behavior Documentation

Test names must be complete sentences describing:
- The **subject** (what is being tested)
- The **action** (what happens)
- The **expected outcome** (what should be true)

Format: `test_{subject}_{action}_{expected_outcome}` or
`test_{condition}_{expected_behavior}`:

```
// ❌ FAIL — names describe implementation, not behavior
test_create_employee()
test_validation()
test_repository()

// ✅ PASS — names document the system's behavior
test_newly_registered_user_receives_a_welcome_email()
test_employee_with_active_contracts_cannot_be_deleted()
test_dashboard_statistics_are_cached_for_five_minutes()
test_cache_is_invalidated_when_an_employee_is_updated()
test_unauthenticated_requests_to_protected_routes_return_401()
test_finance_viewer_cannot_access_user_management_endpoints()
```

### 9.4 Arrange-Act-Assert (AAA) Structure

Every test must have exactly three sections, clearly separated:

```php
public function test_active_employee_count_is_cached_after_first_query(): void
{
    // ARRANGE — set up the world the test operates in
    Cache::flush();
    Employee::factory()->count(5)->create(['status' => 'active']);
    Employee::factory()->count(3)->create(['status' => 'inactive']);

    // ACT — perform the single action under test
    $count = $this->employeeService->getActiveEmployeeCount();

    // ASSERT — verify the expected outcome
    $this->assertEquals(5, $count);
    $this->assertTrue(Cache::has('employees:active_count'));
}
```

Never mix arrange, act, and assert. Never have two `// ACT` sections in one test
(that is two tests in one — split them).

### 9.5 Test Coverage Requirements

| Layer | Minimum Coverage | Reason |
|---|---|---|
| Services (business logic) | 90% | This is where bugs live |
| Repositories (data access) | 80% | Query logic is subtle |
| Controllers (HTTP layer) | 70% | Happy path + auth failures |
| Models (domain logic) | 85% | Computed properties and scopes |
| Jobs (background tasks) | 85% | Including failure paths |
| Utilities / helpers | 95% | Pure functions are easy to test |

**Coverage target ≠ quality target.** 100% coverage with trivial assertions
is worthless. Every assert must verify a behavior that could realistically fail.

---

## SECTION 10 — ARCHITECTURE LAWS

### 10.1 Dependency Rule (Clean Architecture)

Dependencies must always point inward — from the outermost layer (frameworks,
UI, databases) toward the innermost layer (domain entities and business rules).

```
Outer layers:    Controllers → HTTP framework → Database → Third-party libs
Inner layers:    Services → Domain Entities → Business Rules

// RULE: An inner layer must NEVER import from an outer layer
// Domain entities must not know about Laravel, Express, Django, or Eloquent
// Services must not know about HTTP requests or database drivers
```

In practice:
```
// ❌ FAIL — domain entity imports from the HTTP/framework layer
use Illuminate\Http\Request;

class Employee {
    public function updateFromRequest(Request $request): void {} // ← framework dependency
}

// ✅ PASS — service handles framework interaction; entity stays pure
class EmployeeService {
    public function updateEmployee(int $id, UpdateEmployeeDTO $dto): Employee {}
}
```

### 10.2 Don't Repeat Yourself (DRY)

Every piece of knowledge must have a single, authoritative representation
in the system. Duplication is the root of maintenance nightmares: when the
knowledge changes, every copy must be found and updated — and one always gets missed.

DRY applies to:
- **Logic:** the same calculation appearing in two places
- **Configuration:** the same constant defined in two files
- **Structure:** the same query written in two controllers
- **Validation:** the same rule checked in both the controller and the service

```
// ❌ FAIL — probation period defined in two places; will eventually diverge
class EmployeeService {
    public function isOnProbation(Employee $employee): bool {
        return $employee->created_at->diffInDays() < 90;  // 90 hardcoded here
    }
}

class PayrollService {
    public function getProbationaryBonusRate(Employee $employee): float {
        return $employee->created_at->diffInDays() < 90 ? 0.0 : 0.15;  // 90 hardcoded again
    }
}

// ✅ PASS — single source of truth
// config/hr.php:
'probation_period_days' => 90,

// Employee model:
public function isOnProbation(): bool {
    return $this->created_at->diffInDays() < config('hr.probation_period_days');
}

// PayrollService uses $employee->isOnProbation() — no duplication
```

### 10.3 Separation of Concerns

Each layer of the application handles exactly one concern:

| Layer | Concern | Must Not Contain |
|---|---|---|
| Controller | Parse request; call service; return response | Business logic, SQL queries |
| Service | Business rules and orchestration | SQL queries, HTTP knowledge |
| Repository | Data access and query logic | Business rules |
| Model | Domain entity structure and relationships | HTTP knowledge, email sending |
| Job | One async task | Business logic not related to the task |
| Event/Listener | Decouple a trigger from its effects | Direct controller calls |

### 10.4 Principle of Least Surprise

Code must do what its name says — nothing more, nothing less. Surprising
behavior is the fastest path to bugs.

```
// ❌ FAIL — calling a "getter" triggers a database write (deeply surprising)
public function getOrCreateDepartment(string $name): Department {
    return Department::firstOrCreate(['name' => $name]);  // creates a record!
}

// ✅ PASS — the name reveals the side effect
public function findOrCreateDepartment(string $name): Department {}
```

---

## SECTION 11 — THE CLEAN CODE AUDIT CHECKLIST

Run this checklist against every file before delivering it. Every `[ ]` must
become `[✅]`. If a check cannot be completed due to scope, mark it `[⚠️ SMELL]`
and add a `// SMELL:` comment in the code.

### Names
- [ ] Every variable name reveals intent without requiring a comment
- [ ] No disinformation: names match the actual data structure and type
- [ ] No banned words: `data`, `info`, `manager`, `helper`, `utils`, `temp`, `flag`
- [ ] No magic numbers: all constants are named
- [ ] Functions named with verb phrases; classes named with noun phrases
- [ ] Verb prefixes are consistent with the standard table in §1.7

### Functions
- [ ] Every function does one thing (describable without "and")
- [ ] No function exceeds 20 lines (if it does, justify or extract)
- [ ] No function has more than 3 arguments (if 3+, extract a parameter object)
- [ ] No boolean flag arguments
- [ ] No hidden side effects
- [ ] Command-Query Separation observed
- [ ] No error codes returned — typed exceptions thrown instead
- [ ] Try/catch blocks are isolated in their own functions

### Comments
- [ ] No redundant comments (restating what the code says)
- [ ] No commented-out code
- [ ] No journal/timestamp comments
- [ ] All `WHY:` comments explain non-obvious decisions
- [ ] All `WARNING:` comments flag real consequences
- [ ] Every `TODO:` links to a ticket

### Formatting
- [ ] No line exceeds 120 characters
- [ ] No nesting deeper than 3 levels
- [ ] Related code is vertically grouped
- [ ] File reads high-level → low-level from top to bottom
- [ ] Consistent indentation (no mixed tabs/spaces)

### Classes
- [ ] Every class has a single, stateable responsibility
- [ ] No class has more than 10 public methods
- [ ] No constructor has more than 5 dependencies
- [ ] Third-party libraries are wrapped behind interfaces
- [ ] All SOLID principles are satisfied

### Error Handling
- [ ] No `null` returned from functions (Null Object or exception used instead)
- [ ] No `null` passed as a function argument
- [ ] All exceptions are named and typed
- [ ] Third-party exceptions wrapped at the boundary

### Tests
- [ ] Every test satisfies F.I.R.S.T.
- [ ] Every test has exactly one concept
- [ ] Every test follows Arrange-Act-Assert
- [ ] Test names describe behavior, not implementation
- [ ] No test always passes regardless of implementation correctness
- [ ] Coverage meets the minimums in §9.5

---

## SECTION 12 — LANGUAGE-SPECIFIC ADDENDA

### PHP (Laravel)

- All Eloquent queries go in Repository classes — never in Controllers or Blade templates
- `Model::all()` is banned in production code; always use pagination, chunking, or explicit limits
- `env()` calls are banned outside of `config/` files (breaks `config:cache`)
- Mass-assignment: every model declares `$fillable` or `$guarded` explicitly
- `preventLazyLoading(!app()->isProduction())` must be active in AppServiceProvider
- Route closures are banned in production; all routes point to named controller methods
- Request validation belongs in Form Request classes — never inline in controllers

### TypeScript / React

- `any` is banned. Use `unknown` when the type is genuinely unknown; then narrow it.
- Every component has a typed `Props` interface — never use inline or implicit prop types
- API response types are defined in `src/types/` and shared with the API client layer
- `useEffect` with an empty dependency array `[]` must have a `// WHY:` comment explaining why it only runs once
- No `console.log` left in committed code — use a logger abstraction
- Every async function in a component is wrapped in try/catch or handled via the query library's error state

### Python / Django

- Type hints required on every function signature
- `*args` and `**kwargs` in non-decorator, non-utility code require a `# WHY:` comment
- `print()` is banned in production code — use `logging`
- Every model manager with custom querysets is tested independently

### SQL / Migrations

- Every `CREATE TABLE` must have a primary key
- Every foreign key column must have an index defined in the same migration
- No raw string interpolation in queries — parameterized queries only
- Every migration has a tested `down()` method
- Index names must follow the pattern: `idx_{table}_{column(s)}`

---

*This skill encodes Robert C. Martin's Clean Code (2008), The Clean Coder (2011),
and Clean Architecture (2017), translated into enforceable rules for modern stacks.*
*Every rule has a rationale. If a rule seems wrong for the context, record the
exception in DECISIONS.md with justification — do not silently ignore it.*
