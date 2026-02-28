# Composing Software
# That Stays Soft
A Practical Guide to Wiring Independent Tools Together
for the Age of AI-Assisted Development
No orchestration frameworks. No message buses. Just the discipline that makes composed systems changeable.
Contents
## Part One: The Composition Problem
## Chapter 1: Why Independent Tools Are Not Enough
## Chapter 2: The Glue Trap — Where Discipline Goes to Die
## Chapter 3: The Two Kinds of Composition
## Part Two: The Mental Model
## Chapter 4: Flows vs. Plumbing — Extending the Only Rule You Need
## Chapter 5: A Real Example: The Order-to-Invoice Pipeline
## Part Three: The Five Composition Practices
## Chapter 6: Flows Are Your Architecture
## Chapter 7: Result Types at Every Seam
## Chapter 8: Contracts Are Tested, Not Trusted
## Chapter 9: The Dumb Flow Runner
## Chapter 10: Compensation Is Not an Afterthought
## Part Four: Composition Without Pain
## Chapter 11: Why Composed Systems Break
## Chapter 12: The Anti-Corruption Layer (Without the Jargon)
## Chapter 13: Growing a System Flow by Flow
## Part Five: Working With LLMs on Composed Systems
## Chapter 14: Prompt Patterns That Enforce Composition Discipline
## Chapter 15: The Iterative Composition Cycle
## Chapter 16: When to Rewire and When to Rebuild
## Appendix: The Composition Checklist
# Part One
The Composition Problem
## Chapter 1: Why Independent Tools Are Not Enough
The first manifesto — Building Software That Stays Soft — solved the problem of individual tools turning rigid. It taught you to separate decisions from plumbing, enforce file boundaries, and keep your orchestrators dumb. If you followed it, you now have a collection of independent, well-tested tools, each with a clean ToolInput-to-ToolOutput contract.

And then you need two of them to work together.

This is where the first architecture goes silent. Soft Code tells you how to build a brick. It does not tell you how to build a wall. The assumption was always: build good bricks first, figure out the mortar later. That assumption was correct — good bricks are a prerequisite. But the mortar has its own failure modes, its own rigidity traps, and its own discipline. Ignoring it does not make the problem go away. It just delays it until the system is large enough that the pain is real.

Here is what happens in practice. You have an order_quote tool that calculates pricing and an invoice_generator tool that produces PDF invoices. Each works perfectly in isolation. Each has clean contracts, pure decision functions, comprehensive tests. Now you need to pipe the output of one into the input of the other. How hard could it be?

The naive approach is a script:

```python
quote_output = order_quote.run(quote_input)
invoice_input = InvoiceInput(
    customer=quote_output.customer_name,
    line_items=quote_output.items,
    total=quote_output.total_cents,
    tax=quote_output.tax_cents,
)
invoice_output = invoice_generator.run(invoice_input)
```

Six lines. Seems fine. But look at what just happened. You coupled the output shape of order_quote directly to the input shape of invoice_generator. If order_quote renames `customer_name` to `customer`, this script breaks — and so does every other place these tools are composed. You have also skipped every question that matters in production: What happens if order_quote returns an error? What if the data it returns is valid by its own contract but invalid for invoice_generator's expectations? What if you need to add a third tool between them later? What if the composition itself needs to be tested?

That six-line script is the seed of the same concrete that the first manifesto warned you about. Except now it is growing between your tools instead of inside them.

The Core Insight

The first manifesto taught you that coupling inside a tool comes from mixing decisions with plumbing. Coupling between tools comes from a different source: mixing translation with orchestration. Translation is the work of converting one tool's output into another tool's input — adapting shapes, handling mismatches, filling gaps. Orchestration is the work of deciding which tools run in what order and what happens when something fails. When these two concerns share the same code, you get the same rigidity problem you already solved inside individual tools, just at a higher level.

## Chapter 2: The Glue Trap — Where Discipline Goes to Die

There is a well-documented phenomenon in software engineering that researchers call the "glue code problem." The numbers are stark: roughly 70% of development effort in integrated systems goes to glue code — the connective tissue between components. This is not because glue is inherently hard. It is because glue is treated as unimportant.

Glue code is the code nobody plans. Nobody designs it. Nobody tests it rigorously. Nobody puts it through architectural review. It is "just a few lines here and a few lines there." And then one day you look at your system and the glue is the system. The tools themselves, carefully architected with Soft Code discipline, account for maybe 30% of the total codebase. The other 70% is ad-hoc wiring that nobody owns.

This is especially dangerous with LLM-assisted development. An LLM asked to "wire tool A to tool B" will produce exactly the kind of naive script shown above. It optimizes for making the connection work right now. It does not think about what happens when tool A's contract changes, or when you need to insert tool C between them, or when you need to run the same composition with different error handling policies in different contexts.

The Quadratic Problem

Here is the mathematical reality. If you have N tools that need to interact with each other, and each interaction requires custom glue code, you have O(N squared) pieces of glue. With five tools, that is twenty-five potential connection points. With ten tools, one hundred. With twenty, four hundred. The glue overwhelms the tools, and you only notice when it is far too late to do anything about it.

The solution is not to avoid glue. Glue is necessary and unavoidable. The solution is to treat glue with the same architectural discipline you apply to tools. The first manifesto gave you practices for building tools that stay soft. This manifesto gives you practices for composing tools that stay soft.

## Chapter 3: The Two Kinds of Composition

Just as all code is either decisions or plumbing, all composition falls into one of two categories. Understanding which one you are dealing with determines everything about how you structure it.

Sequential composition is when tools run one after another, each consuming the output of the previous one. This is a pipeline: tool A produces data, tool B transforms it, tool C outputs the result. Most business workflows are sequential at their core: receive an order, calculate pricing, generate an invoice, send a notification.

Parallel composition is when multiple tools can run independently on the same or different inputs, and their outputs are combined afterward. This is a fan-out/fan-in: given a customer record, simultaneously check credit, validate address, and verify inventory, then combine the results to decide whether to approve the order.

There is a third pattern that looks like it should be a separate category but is not: conditional composition. "If the credit check fails, run the manual review tool instead of the auto-approval tool." This is not a distinct kind of composition. It is a decision, and decisions belong in decision functions — not in the composition layer. The flow runner should be as dumb as the orchestrator inside a single tool. Push the conditional logic into a decision function that examines the output of the credit check and returns which path to take. The flow runner just follows the path.

This parallels the single-tool rule perfectly. Inside a tool, the orchestrator has zero branching. Between tools, the flow runner has zero branching. All intelligence lives in decision functions, whether they belong to a single tool or to the composition layer.

# Part Two
The Mental Model
## Chapter 4: Flows vs. Plumbing — Extending the Only Rule You Need

The first manifesto gave you one rule: every function should either make a decision or perform plumbing, never both.

The composition manifesto extends it: every piece of composition code should either define a flow or translate data, never both.

A flow definition describes which tools run in what order, what data passes between them, and what happens when a step fails. It is the composition equivalent of the dumb orchestrator. It reads like a recipe. It contains no logic about how to convert one tool's output into another tool's input. It just says: "step 1 feeds into step 2, step 2 feeds into step 3."

A translator converts one tool's output shape into another tool's input shape. It is a pure function — data in, data out. No side effects, no I/O, no decisions about flow control. It just maps fields, fills defaults, transforms formats.

Here is what this looks like in practice:

```python
# TRANSLATOR — pure function, no flow logic
def quote_to_invoice_input(quote: QuoteOutput) -> InvoiceInput:
    return InvoiceInput(
        customer=quote.customer_name,
        line_items=[
            InvoiceLineItem(
                description=item.product_name,
                quantity=item.qty,
                unit_price_cents=item.unit_price_cents,
                total_cents=item.line_total_cents,
            )
            for item in quote.items
        ],
        subtotal_cents=quote.subtotal_cents,
        tax_cents=quote.tax_cents,
        total_cents=quote.total_cents,
    )
```

```python
# FLOW DEFINITION — sequence only, no translation logic
order_to_invoice_flow = Flow(
    name="order_to_invoice",
    steps=[
        Step(tool="order_quote", input_type=QuoteInput),
        Step(tool="invoice_generator",
             input_type=InvoiceInput,
             translator=quote_to_invoice_input),
    ],
)
```

The translator is independently testable: give it a QuoteOutput, verify you get the right InvoiceInput. The flow definition is independently readable: anyone can see the sequence without wading through field-mapping code. And when order_quote changes its output shape, exactly one function needs to change: the translator. The flow, the tools, and every other translator are untouched.

This is the same principle as the first manifesto, applied one level up. Separation of concerns is fractal. It applies within functions, within files, within tools, and between tools.

## Chapter 5: A Real Example — The Order-to-Invoice Pipeline

Let us build a real composed system from start to finish. We have three existing Soft Code tools:

- **order_quote**: takes order details, returns pricing with discounts and tax
- **credit_check**: takes a customer ID, returns a credit decision (approve/deny/review)
- **invoice_generator**: takes validated order details, returns a rendered invoice

Each tool was built following the first manifesto. Each has a contracts.py with ToolInput, ToolOutput, and a run() function. Each is thoroughly tested in isolation.

Now we need to compose them into an order processing pipeline: receive an order, calculate its pricing, check the customer's credit, and if approved, generate an invoice.

Step 1: Identify the translation boundaries.

Before writing any code, map out where data shape changes happen:

- order_quote.ToolOutput must become credit_check.ToolInput (we need the customer ID and the total amount for the credit decision)
- order_quote.ToolOutput must also become invoice_generator.ToolInput (we need line items, totals, tax)
- credit_check.ToolOutput must inform whether we proceed to invoicing (this is a decision, not a translation)

Step 2: Identify the decisions that belong to the composition layer.

These are decisions that no single tool owns. They belong to the flow itself:

- Should we proceed to invoicing? This depends on the credit check result, but the credit_check tool does not know about invoicing. The decision "approved means proceed, denied means stop, review means queue for manual processing" belongs to the composition layer.
- What happens if credit check fails with an error (not a denial, but an actual system error)? Retry? Proceed with a flag? Stop everything?

These decisions get their own decision functions, just like decisions inside a tool. They are pure functions. They live in their own file. They are tested.

```python
# flow_decisions.py — pure, no I/O

@dataclass
class FlowDecision:
    proceed: bool
    reason: str
    requires_review: bool = False

def should_proceed_to_invoice(credit_result: CreditOutput) -> FlowDecision:
    if credit_result.decision == "approved":
        return FlowDecision(proceed=True, reason="credit_approved")
    if credit_result.decision == "review":
        return FlowDecision(proceed=False, reason="manual_review_required",
                          requires_review=True)
    return FlowDecision(proceed=False, reason=f"credit_denied: {credit_result.reason}")
```

Step 3: Write the translators.

Each translator is a pure function in its own file. Each maps one tool's output to another tool's input. Each is tested independently.

```python
# translators.py — pure functions, no I/O, no flow logic

def quote_to_credit_input(quote: QuoteOutput) -> CreditInput:
    return CreditInput(
        customer_id=quote.customer_id,
        requested_amount_cents=quote.total_cents,
    )

def quote_to_invoice_input(quote: QuoteOutput) -> InvoiceInput:
    return InvoiceInput(
        customer=quote.customer_name,
        line_items=[...],  # field mapping
        subtotal_cents=quote.subtotal_cents,
        tax_cents=quote.tax_cents,
        total_cents=quote.total_cents,
    )
```

Step 4: Define the flow.

The flow runner is the equivalent of the dumb orchestrator. It calls translators and tools in sequence. It does not contain any business logic. Conditional paths are resolved by calling decision functions, not by embedding if-statements in the flow.

```python
# order_processing_flow.py — the dumb flow runner

def run_order_flow(order_input: QuoteInput) -> FlowResult:
    quote_result = run_step("order_quote", order_input)
    credit_input = quote_to_credit_input(quote_result.value)
    credit_result = run_step("credit_check", credit_input)
    decision = should_proceed_to_invoice(credit_result.value)
    invoice_input = quote_to_invoice_input(quote_result.value)
    invoice_result = run_step("invoice_generator", invoice_input)
    return FlowResult.success(invoice=invoice_result.value)
```

Wait — there is a problem. This flow has an implicit branch: if the credit decision says "do not proceed," we should not run the invoice generator. But we said the flow runner has zero branching.

Here is how you handle it. The flow runner does not branch. The decision function produces a value. The flow runner passes that value to the next step. The next step either acts or no-ops based on the value. In practice, this means wrapping the flow in a result type that short-circuits on failure:

```python
def run_order_flow(order_input: QuoteInput) -> FlowResult:
    return (
        run_step("order_quote", order_input)
        .then(quote_to_credit_input)
        .run_step("credit_check")
        .then(lambda credit: check_proceed(credit, quote_result))
        .then(quote_to_invoice_input)
        .run_step("invoice_generator")
    )
```

This is railway-oriented programming. The flow stays on the success track until something fails or a decision diverts it to the failure track. No if-statements. No branching. Just a chain that short-circuits.

Step 5: Test the composition.

You already have unit tests for each tool. Now you need tests for the composition layer:

- Test each translator: given this QuoteOutput, do I get the expected CreditInput?
- Test each flow decision: given an approved CreditOutput, does should_proceed_to_invoice return proceed=True?
- Test the flow end-to-end: given a complete QuoteInput, does the flow produce the expected InvoiceOutput? (This can use the real tool run() functions since they are pure.)

The beautiful thing about Soft Code tools is that end-to-end flow tests are fast and require no infrastructure. Every tool's run() function is pure computation. No databases, no network, no files. You can run the entire flow in a unit test.

# Part Three
The Five Composition Practices
## Chapter 6: Flows Are Your Architecture

Just as "files are your architecture" inside a tool, flows are your architecture between tools.

Every composed workflow gets its own directory:

```
flows/
    order_to_invoice/
        DECISIONS.md          # What decisions does this flow make?
        models.py             # FlowResult, FlowDecision, any flow-specific data
        translators.py        # Pure functions mapping tool outputs to tool inputs
        decisions.py          # Pure functions for flow-level decisions
        flow.py               # The dumb flow runner
        compensation.py       # What to undo when steps fail
        tests/
            test_translators.py
            test_decisions.py
            test_flow.py
```

This is the same structure as a Soft Code tool, but for composition. The flow is the orchestrator. The translators are the plumbing. The decisions are the decision functions. The compensation is new — we will cover it in Chapter 10.

Notice that the flow directory does not contain any tool code. Tools stay in core/. The flow directory only contains the wiring: how tools connect, how data translates between them, and what flow-level decisions are made. If you need to change a tool's logic, you go to core/. If you need to change how tools are wired together, you go to flows/.

This physical separation is just as important as the separation inside tools. It prevents the most common failure mode of composed systems: flow logic leaking into tools, or tool logic leaking into flows.

The Flow Registry

Just as tools have tools_registry.json, flows have a registry:

```json
{
    "flows": [
        {
            "name": "order_to_invoice",
            "description": "Processes an order through pricing, credit check, and invoice generation",
            "status": "active",
            "tools": ["order_quote", "credit_check", "invoice_generator"],
            "entry_point": "flows/order_to_invoice/flow.py"
        }
    ]
}
```

This registry is the single source of truth for what compositions exist, which tools they use, and where to find them. It enables automated dependency tracking: if order_quote changes its contract, you can programmatically find every flow that uses it.

## Chapter 7: Result Types at Every Seam

Inside a Soft Code tool, functions communicate through plain dataclasses. Between tools in a composed flow, functions communicate through Result types — dataclasses that explicitly represent success or failure.

Here is why this matters. When you call a single function, Python's exception mechanism works fine. If the function raises, the caller catches. The control flow is clear. But when you compose a chain of tools with translators between them, exceptions become a problem. An exception in step 3 of a five-step flow unwinds the entire stack. You lose context about which step failed, what the partial results were, and what compensation actions you might need to take.

Result types solve this by making success and failure explicit values that flow through the pipeline:

```python
@dataclass
class Success:
    value: any
    step: str      # which step produced this

@dataclass
class Failure:
    error: str
    step: str      # which step failed
    partial: dict  # results from completed steps

Result = Success | Failure
```

Every step in a flow returns a Result. Every translator accepts a Success value and returns a Result. The flow runner chains Results. If any step returns a Failure, the chain short-circuits — subsequent steps do not run, and the failure propagates with full context about where it happened and what succeeded before it.

This is the railway pattern. Your flow has two tracks: the success track where everything proceeds normally, and the failure track where the flow has been diverted. The flow runner just follows the track. It does not decide what to do about failures. That is a decision, and decisions belong in decision functions.

```python
# The Result type — minimal, no dependencies
@dataclass(frozen=True)
class Ok:
    value: object

@dataclass(frozen=True)
class Err:
    error: str
    context: dict

def bind(result, fn):
    """Chain a function onto a result. Short-circuit on error."""
    if isinstance(result, Err):
        return result
    try:
        return Ok(fn(result.value))
    except Exception as e:
        return Err(str(e), {"input": result.value})
```

You do not need a library for this. The entire Result type is ten lines. The bind function is six lines. This is not monadic ceremony — it is a dataclass and a function. Any LLM can understand it, test it, and work with it.

The Rule

Every tool-to-tool boundary uses Result types. No exceptions cross tool boundaries. If a tool raises internally, the flow runner catches it at the boundary and wraps it in an Err. The flow never sees a raw exception — only Results.

## Chapter 8: Contracts Are Tested, Not Trusted

Inside a tool, you trust that your own functions return the right shapes. Between tools, you do not trust — you verify.

This is the composition equivalent of the first manifesto's "tests are your first interface." But composition testing has a different character. Inside a tool, you test that functions produce correct outputs for given inputs. Between tools, you test that tool A's output satisfies tool B's expectations — even though A and B know nothing about each other.

This is contract testing, and it is the single most important practice for keeping composed systems soft.

The Three Levels of Composition Testing

Level 1: Translator tests. For each translator function, verify that a known tool output produces the expected tool input. These are pure unit tests, fast and deterministic.

```python
def test_quote_to_invoice_input():
    quote = QuoteOutput(
        customer_name="Acme Corp",
        items=[QuoteItem(product_name="Widget", qty=10,
                        unit_price_cents=1000, line_total_cents=10000)],
        subtotal_cents=10000,
        tax_cents=800,
        total_cents=10800,
    )
    result = quote_to_invoice_input(quote)
    assert result.customer == "Acme Corp"
    assert result.total_cents == 10800
    assert len(result.line_items) == 1
```

Level 2: Contract compatibility tests. For each pair of connected tools, verify that the real output of tool A (from its own tests) can pass through the translator and produce valid input for tool B. This catches the subtle shape mismatches that cause runtime failures.

```python
def test_quote_output_compatible_with_invoice_input():
    # Run tool A with a known input
    quote_result = order_quote.run(sample_quote_input())
    # Translate
    invoice_input = quote_to_invoice_input(quote_result)
    # Verify tool B accepts it (does not raise)
    invoice_result = invoice_generator.run(invoice_input)
    assert isinstance(invoice_result, InvoiceOutput)
```

Level 3: Flow integration tests. Run the entire flow end-to-end with realistic inputs and verify the final output. Because all Soft Code tools are pure, these tests require no infrastructure and run in milliseconds.

```python
def test_full_order_to_invoice_flow():
    result = run_order_flow(sample_order_input())
    assert isinstance(result, Ok)
    assert result.value.invoice.total_cents == expected_total
```

The Contract Compatibility Catch

Here is the subtle problem that Level 2 tests catch. Suppose order_quote always returns amounts in cents (integers), but invoice_generator expects amounts in dollars (floats). Both tools work perfectly in isolation. Their own tests pass. But when composed, the translator silently passes integer cents where float dollars are expected, and the invoice shows $10800 instead of $108.00.

This is the impedance mismatch problem — the most common failure mode in composed systems. It is not a bug in either tool. It is a bug in the assumption that their contracts are compatible. Level 2 tests catch this by running real tool outputs through real translators and feeding them to real tool inputs.

When Contract Changes Happen

When a tool changes its contract (ToolInput or ToolOutput shape), the contract compatibility tests for every flow that uses that tool will fail. This is the desired behavior. The test failures tell you exactly which translators need updating and which flows are affected. Without these tests, you discover the breakage in production.

The validation scripts from the first manifesto can be extended to enforce this:

- Every flow must have translator tests (Level 1)
- Every tool-to-tool connection must have compatibility tests (Level 2)
- Every flow must have at least one end-to-end test (Level 3)

## Chapter 9: The Dumb Flow Runner

The flow runner is the between-tools equivalent of the within-tool orchestrator. It has the same character: it should be dumb. It should contain no logic. It should read like a recipe.

The flow runner's only job is to execute steps in order, pass Results between them, and stop when something fails. It does not decide what to do about failures. It does not translate data between tools. It does not contain business logic. It just runs the sequence.

```python
def run_order_flow(order_input):
    steps = [
        ("order_quote", order_quote.run, None),
        ("credit_check", credit_check.run, quote_to_credit_input),
        ("invoice_gen", invoice_generator.run, quote_to_invoice_input),
    ]

    results = {}
    for name, tool_run, translator in steps:
        source = results.get(prev_step)
        input_data = translator(source) if translator else order_input
        result = safe_run(name, tool_run, input_data)
        if isinstance(result, Err):
            return FlowResult.failed(step=name, error=result, completed=results)
        results[name] = result.value

    return FlowResult.success(results)
```

Count the lines. Under twenty. No if-else branching on business conditions. No field mapping. No error handling decisions. The flow runner is a for-loop over a list of steps. That is it.

If you find your flow runner growing beyond thirty lines, something is wrong. Either business logic has leaked in (extract it to a decision function), or translation logic has leaked in (extract it to a translator), or error handling logic has leaked in (extract it to a compensation handler).

The Constraints

Mirroring the first manifesto's orchestrator constraints:

- Maximum 40 lines
- No business logic (no decisions about data values)
- No data translation (no field mapping or type conversion)
- No error recovery logic (no retry, no fallback, no compensation)
- The only control flow allowed is the result-chain short-circuit

These constraints are mechanically enforceable with the same kind of AST-based validation scripts used in Soft Code. A flow runner that violates them will be caught at preflight.

## Chapter 10: Compensation Is Not an Afterthought

When a single Soft Code tool fails, the damage is contained. The tool is pure — it has no side effects. A failed calculation just returns an error. There is nothing to undo.

But when a composed flow fails partway through, the earlier steps may have already produced results that downstream systems depend on. If step 3 of a five-step flow fails, steps 1 and 2 have already completed. Their results may have been recorded, transmitted, or acted upon by external systems through adapters. This is the partial failure problem, and it is the hardest problem in composition.

The saga pattern addresses this with compensating transactions: for every forward action, define a reverse action that logically undoes it. If step 3 fails, run the compensation for step 2, then the compensation for step 1, in reverse order.

For Soft Code tools, compensation has a unique advantage. Because tools are pure functions with no side effects, the tools themselves never need compensation. Only the adapters — the thin plumbing wrappers that interact with the outside world — can produce effects that need undoing. This dramatically simplifies the compensation problem.

```python
# compensation.py — what to undo when the flow fails

@dataclass
class CompensationStep:
    step_name: str
    compensate: Callable  # function to undo this step's side effects

def compensate_order_flow(completed_steps, failed_step):
    """Run compensations in reverse order for completed steps."""
    compensations = {
        "send_invoice": lambda ctx: void_invoice(ctx["invoice_id"]),
        "charge_payment": lambda ctx: refund_payment(ctx["payment_id"]),
        "reserve_inventory": lambda ctx: release_inventory(ctx["reservation_id"]),
    }

    for step_name in reversed(completed_steps):
        if step_name in compensations:
            compensations[step_name](completed_steps[step_name])
```

Notice: the compensations are for adapter-level effects (sending an invoice, charging a payment, reserving inventory), not for tool-level computations. The pure tool outputs — calculated prices, generated invoice data — need no compensation. They are just data. Data does not need to be "undone."

The Compensation Rules

1. Every flow that triggers side effects (through adapters) must define compensations for those effects.
2. Compensations must be idempotent: running them twice produces the same result as running them once.
3. Compensations are defined alongside the flow, in compensation.py, not scattered across adapter code.
4. The flow runner does not execute compensations directly. It reports the failure with enough context for the compensation handler to act.

When Pure Tools Make Compensation Simple

Here is the deep payoff of the Soft Code approach. In a typical composed system, every step might write to a database, call an API, or modify shared state. Compensation for a five-step flow means defining five undo operations, each of which must handle partial states, concurrent modifications, and idempotency.

In a Soft Code composed system, the tools are pure. They compute results. They do not write to databases or call APIs. Only the adapters at the edges do. So a five-step flow where three steps are pure computations and two steps are adapter calls only needs two compensations, not five. The pure steps are free — they produced data, not effects, and data needs no undoing.

This is why the first manifesto's insistence on "side effects at the edges" pays compound interest in composition. The less effectful your system is, the less compensation you need. And the less compensation you need, the more confidently you can compose tools into complex flows without fearing partial failure.

# Part Four
Composition Without Pain
## Chapter 11: Why Composed Systems Break

Composed systems break for reasons that are fundamentally different from why individual tools break. Understanding these failure modes is the first step to preventing them.

Failure Mode 1: Shape Mismatch

Tool A returns an amount in cents. Tool B expects an amount in dollars. Both tools are correct. The translator between them is where the bug lives — or more precisely, where the bug hides, because nobody tested the translator with real tool outputs.

Prevention: Level 2 contract compatibility tests (Chapter 8).

Failure Mode 2: Semantic Mismatch

Tool A returns a "status" field with values "active," "inactive," "pending." Tool B accepts a "status" field with values "enabled," "disabled," "waiting." The shapes match (both are strings), but the meanings do not.

Prevention: Translators must handle semantic mapping, not just structural mapping. Test translators with the full range of possible values, not just the happy path.

Failure Mode 3: The N-Squared Problem

You add a new tool. It needs to interact with three existing tools. That is three new translators, three sets of contract tests, and potentially three new flows. Adding the next tool requires four translators. The effort of adding tools grows quadratically.

Prevention: Shared data shapes. If multiple tools operate in the same domain (e.g., pricing), define a shared vocabulary — a set of dataclasses that multiple tools use. This is not coupling. It is a deliberate, versioned, tested shared contract. When tools agree on the shape of "money" or "line item" or "customer," translators between them become trivial or disappear entirely.

Failure Mode 4: Cascade Failures

Tool A's contract changes. This breaks the translator for flow X. A developer fixes the translator for flow X but does not realize flow Y also uses tool A through a different translator. Flow Y breaks in production.

Prevention: The flow registry (Chapter 6) combined with contract compatibility tests. When tool A changes, the CI pipeline finds every flow that uses tool A and runs their contract tests. All breakages surface simultaneously, not one at a time in production.

Failure Mode 5: Ghost Dependencies

A flow works because two tools happen to share an implicit assumption — for example, both assume dates are in UTC, or both assume amounts are in cents. This assumption is not documented, not tested, and not visible in any translator. When one tool changes its assumption, the flow breaks in a way that is extremely difficult to diagnose.

Prevention: Make all assumptions explicit in the translator. If tool A outputs UTC timestamps and tool B expects UTC timestamps, the translator should still explicitly document and test this: assert that the timestamp is UTC, and convert it if it is not. The translator is the place where implicit assumptions become explicit contracts.

## Chapter 12: The Anti-Corruption Layer (Without the Jargon)

In domain-driven design, there is a concept called the "anti-corruption layer." The name is dramatic but the idea is simple: when two systems with different worldviews need to communicate, put a translation layer between them so that neither system's concepts leak into the other.

In the Soft Code composition framework, the translator IS the anti-corruption layer. Each tool has its own language — its own data shapes, its own field names, its own conventions for representing money or dates or status values. The translator protects each tool from the other's language.

The key discipline is: translators must translate in one direction only. A translator converts order_quote output to credit_check input, never the reverse. If you need the reverse direction, write a separate translator. This prevents the common mistake of creating "shared models" that try to serve both tools and end up coupling them.

```
Tool A ──→ [Translator A→B] ──→ Tool B
Tool B ──→ [Translator B→A] ──→ Tool A  (separate function!)
```

One-way translators are simpler, easier to test, and easier to replace. They are also easier for an LLM to write correctly, because each translator has exactly one input type and one output type. No ambiguity about which direction the data flows.

When to Use a Shared Model

Despite the one-way translator principle, there are cases where a shared model makes sense. Specifically: when multiple tools in the same domain genuinely share a concept that is fundamental to the domain, not just incidentally similar.

Money is a good example. If five tools all deal with monetary amounts, having each tool define its own MoneyAmount dataclass and writing translators between them is wasteful. A shared `core/common/money.py` with a single Money dataclass that all five tools import reduces translator count and eliminates the risk of subtle differences in how tools represent money.

The test for whether a shared model is appropriate: would changing this dataclass require changing the business logic in multiple tools? If yes, it is a genuine shared concept and should be shared. If no — if each tool would just need its translator updated — keep them separate.

## Chapter 13: Growing a System Flow by Flow

A well-structured composed system grows by adding flows, not by making existing flows bigger. This mirrors the first manifesto's principle of growing by adding files, not by making files bigger.

Here is what healthy growth looks like:

Month 1: Build the order_to_invoice flow (three tools, two translators).

Month 2: Need to add payment processing. Do not modify the order_to_invoice flow. Instead, build a new flow: invoice_to_payment (two tools, one translator). The order_to_invoice flow is untouched.

Month 3: Need an end-to-end flow from order to payment. Build order_to_payment, which composes the two existing flows. It calls order_to_invoice, then invoice_to_payment. It adds one new translator (invoice output to payment input) and one new set of tests. The existing flows are untouched.

Month 4: Need a returns/refund workflow. Build a new flow: process_refund (three tools, two translators, two compensations). It references the payment tool but through its own translator. The order flows are untouched.

Each month adds a directory under flows/. Existing flows are never modified to accommodate new flows. This is the same "growth by addition" principle, applied at the composition level.

When flows compose other flows, you get a hierarchy:

```
flows/
    order_to_invoice/          # low-level flow
    invoice_to_payment/        # low-level flow
    order_to_payment/          # composes the above two
    process_refund/            # independent flow
```

The top-level flow (order_to_payment) is itself a dumb flow runner. It does not contain the logic of its sub-flows. It just calls them in sequence with translators between them. The same principles apply at every level: dumb flow runners, pure translators, explicit result types, tested contracts.

# Part Five
Working With LLMs on Composed Systems
## Chapter 14: Prompt Patterns That Enforce Composition Discipline

Composing tools with LLMs has different pitfalls than building tools with LLMs. Here are prompt patterns that consistently produce well-structured compositions.

Pattern 1: The Translator Isolation Prompt

Never ask an LLM to "connect tool A to tool B." It will produce a coupled script. Instead, ask for the translator in isolation:

"Here is the ToolOutput dataclass from order_quote: [paste it].
Here is the ToolInput dataclass from invoice_generator: [paste it].
Write a pure function called quote_to_invoice_input that converts the first into the second. Constraints:
- No imports except the two dataclasses
- No I/O
- No side effects
- Handle the case where optional fields are None
Include tests."

Pattern 2: The Flow Skeleton Prompt

Ask for the flow structure before any implementation:

"I need to compose these tools into a flow: order_quote, credit_check, invoice_generator.
Give me:
1. The list of translators needed (which tool output maps to which tool input)
2. The flow-level decisions needed (decisions that no single tool owns)
3. The flow sequence (which tool runs after which)
4. The compensation actions needed
Do not write any code yet."

Pattern 3: The Contract Test Prompt

After translators exist, ask for contract tests explicitly:

"Here is my translator function quote_to_invoice_input.
Here is a sample QuoteOutput from the order_quote test suite.
Write a test that:
1. Passes the sample through the translator
2. Verifies the result is a valid InvoiceInput
3. Checks that all amounts are correctly mapped
4. Checks edge cases: zero quantities, empty descriptions, maximum values"

Pattern 4: The Composition Review Prompt

Before considering a flow complete, ask for a review:

"Review this flow for composition problems:
1. Are there any implicit assumptions between tools that are not captured in translators?
2. Is there any business logic in the flow runner that should be in a decision function?
3. Are there any data shape mismatches that could cause runtime errors?
4. Is compensation needed for any step? If so, is it defined?"

## Chapter 15: The Iterative Composition Cycle

Here is the cycle that produces well-structured compositions. It assumes the tools already exist and are Soft Code compliant.

Phase 1: Map the connections (10 minutes, no code)

On paper or in your head, draw the tools as boxes and the data flows as arrows. For each arrow, note: what shape goes in, what shape comes out, what translation is needed.

Phase 2: Identify flow-level decisions (5 minutes, no code)

List the decisions that belong to the composition, not to any individual tool. These are the decisions about whether to proceed, what to do on failure, and how to combine results from parallel tools.

Phase 3: Write translators with tests (20 minutes, LLM-assisted)

One translator per tool-to-tool connection. Test each with real sample data from the upstream tool's test suite. Do not move on until every translator has passing tests with realistic data.

Phase 4: Write flow decisions with tests (10 minutes, LLM-assisted)

One decision function per flow-level decision. These are small — usually under ten lines — because they operate on tool outputs, not on raw data.

Phase 5: Write the flow runner (5 minutes)

Wire it together. This should be under thirty lines. If it is more, something has leaked in.

Phase 6: Write contract compatibility tests (15 minutes)

For each tool-to-tool connection, run the upstream tool with sample input, pass the output through the translator, feed it to the downstream tool, and verify success.

Phase 7: Define compensations (10 minutes, if applicable)

For any step that triggers side effects through adapters, define the compensation action. Only adapter-level effects need compensation — pure tool computations do not.

Total time: about 75 minutes for a three-tool flow. This scales roughly linearly: a five-tool flow takes about two hours, not quadratic time, because the structure is predictable and each piece is independent.

## Chapter 16: When to Rewire and When to Rebuild

Even with good practices, compositions can degrade. Here is how to diagnose and respond.

Rewire when: a translator has grown beyond thirty lines and contains conditional logic. This means the tools' contracts have diverged enough that the translation is becoming a transformation. Extract the complex mapping into a dedicated "adapter tool" — a lightweight Soft Code tool whose sole job is data transformation — and simplify the translator to just call it.

Rewire when: the flow runner has grown beyond forty lines. This means business logic or error handling has leaked in. Extract it into decision functions or compensation handlers.

Rebuild the flow when: the sequence of tools no longer matches the business process. If the business says "we now check credit before calculating pricing, not after," you should build a new flow with the new sequence, not modify the existing one. The old flow can be deprecated. The tools themselves are unchanged.

Do not rebuild when: a single translator needs updating because a tool changed its contract. That is a normal, healthy change — exactly what the translator layer is designed to absorb.

Do not rebuild when: you need to add a step to a flow. Instead, create a new flow that composes the existing flow with the additional step. Growth by addition, not modification.

The key insight: translators are the shock absorbers of the composition layer. When tools change, translators change. When business processes change, flows change. When both change at the same time, you might need a new flow with new translators. But the tools themselves should rarely change because of composition needs — if they do, the composition layer is not doing its job.

# Appendix
The Composition Checklist

Use this checklist every time you compose tools into a flow. It complements the original Soft Code checklist, which you should still use for each individual tool.

Before Writing Any Composition Code

☐ Can I draw the tools as boxes and the data flows as arrows?
☐ For each arrow, do I know what shape goes in and what shape comes out?
☐ Have I identified which decisions belong to the composition layer (not to any tool)?
☐ Have I identified which steps produce side effects that might need compensation?

When Writing Translators

☐ Is each translator a pure function with no I/O and no side effects?
☐ Does each translator go in one direction only (A→B, never B→A in the same function)?
☐ Does each translator handle edge cases (None values, empty collections, boundary values)?
☐ Does each translator have tests using realistic data from the upstream tool's test suite?

When Writing the Flow Runner

☐ Is the flow runner under 40 lines?
☐ Does it contain zero business logic? (No decisions about data values)
☐ Does it contain zero translation logic? (No field mapping)
☐ Does it use Result types at every tool boundary?
☐ Does it read like a recipe — just a sequence of steps?

When Writing Flow Decisions

☐ Is each decision a pure function?
☐ Is each decision tested independently of the flow?
☐ Are decisions in decisions.py, not embedded in the flow runner?

When Testing the Composition

☐ Level 1: Does each translator have unit tests?
☐ Level 2: Does each tool-to-tool connection have contract compatibility tests?
☐ Level 3: Does the flow have at least one end-to-end test?
☐ Do contract tests use real tool outputs, not hand-crafted mocks?

When Handling Failures

☐ Does the flow use Result types (not exceptions) at tool boundaries?
☐ Are compensations defined for every step that produces side effects?
☐ Are compensations idempotent?
☐ Are compensations tested?

When Adding to an Existing System

☐ Am I adding a new flow directory, or modifying an existing flow?
☐ If composing existing flows, am I creating a new higher-level flow (not modifying the sub-flows)?
☐ Have I updated the flow registry?
☐ Do existing flow tests still pass without modification?

Remember: the tools are the bricks. The flows are the walls. The translators are the mortar. Each material has its own discipline. A well-built wall requires well-made bricks, properly mixed mortar, and careful placement. Skip any one of these and the wall will not hold.
