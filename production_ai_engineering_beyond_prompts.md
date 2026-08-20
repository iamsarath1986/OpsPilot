# Production AI Engineering: Beyond Prompt Engineering

Yes. What she described is one of the biggest realizations people have
when moving from "prompt engineering" to building production AI systems.

A lot of beginners think:

``` text
User Input
    ↓
Prompt
    ↓
LLM
    ↓
Answer
```

But production AI systems usually look more like:

``` text
User Input
    ↓
Input Validation
    ↓
Guardrails
    ↓
Prompt Construction
    ↓
LLM
    ↓
Output Validation
    ↓
Fallback Logic
    ↓
Monitoring & Logging
    ↓
Final Response
```

## 1. Prompts are only a small piece

A prompt tells the model what you want.

Example:

``` text
Extract customer information from this email.
Return JSON with:
{
  "name": "",
  "email": "",
  "phone": ""
}
```

This works most of the time.

But in production, "most of the time" is not enough.

What happens when the model returns:

``` json
{
  "customer_name": "John"
}
```

instead of the expected schema?

Your application breaks.

That's why prompting alone isn't sufficient.

------------------------------------------------------------------------

## 2. Guardrails

Guardrails prevent bad inputs and dangerous outputs.

### Input Guardrails

Before sending data to the LLM:

``` python
if len(user_message) > 10000:
    reject()

if contains_prompt_injection():
    block()
```

Examples:

-   Prompt injection detection
-   PII filtering
-   Toxic content filtering
-   Length limits
-   Rate limiting

### Output Guardrails

After the LLM responds:

``` python
if contains_credit_card():
    remove()
```

Examples:

-   Prevent hallucinated actions
-   Remove sensitive data
-   Detect toxic content
-   Ensure policy compliance

Popular tools:

-   Guardrails AI --- https://www.guardrailsai.com
-   Langfuse --- https://langfuse.com (monitoring)
-   OpenAI Safety Best Practices ---
    https://platform.openai.com/docs/guides/safety-best-practices

------------------------------------------------------------------------

## 3. Structured Output

One of the most important production techniques.

Instead of:

``` text
Summarize this invoice.
```

You ask for:

``` json
{
  "invoice_number": "",
  "amount": 0,
  "due_date": ""
}
```

Then validate it.

Example:

``` python
class Invoice(BaseModel):
    invoice_number: str
    amount: float
    due_date: str
```

If validation fails:

``` python
try:
    Invoice.model_validate(response)
except:
    retry()
```

This is how modern AI applications become reliable.

Many companies now prefer:

``` text
LLM → JSON Schema → Validation
```

instead of free-text outputs.

------------------------------------------------------------------------

## 4. Validation Layers

Even if the model returns valid JSON, the values may be wrong.

Example:

``` json
{
  "email": "hello"
}
```

Schema validation passes?

Maybe.

Business validation fails.

So you add another layer:

``` python
if not valid_email(email):
    reject()
```

Production systems often have:

### Layer 1

Syntax validation

``` text
valid JSON
```

### Layer 2

Schema validation

``` text
required fields exist
```

### Layer 3

Business validation

``` text
email format correct
date not in past
price > 0
```

------------------------------------------------------------------------

## 5. Fallback Mechanisms

LLMs fail.

The question is not "if."

The question is "when."

Example:

### Primary Model

``` text
GPT-5
```

If timeout:

``` text
Claude
```

If timeout:

``` text
Cached Answer
```

If timeout:

``` text
Human Review
```

Example architecture:

``` text
User
 ↓
GPT-5
 ↓ fail
Claude
 ↓ fail
Mistral
 ↓ fail
Human Queue
```

------------------------------------------------------------------------

## 6. Retrieval Validation (RAG)

Many AI engineers spend more time validating retrieved data than writing
prompts.

Example:

``` text
Question:
What is our refund policy?
```

Instead of trusting the model:

``` text
Search company documents
↓
Retrieve top documents
↓
Check relevance score
↓
Send to LLM
↓
Generate answer
```

Most hallucination problems are actually retrieval problems.

------------------------------------------------------------------------

## 7. Monitoring and Evaluation

Real AI products continuously measure:

-   Accuracy
-   Latency
-   Cost
-   Hallucination rate
-   User satisfaction

Example:

``` text
Prompt Version A
Accuracy: 82%

Prompt Version B
Accuracy: 91%
```

This is why AI Engineers spend significant time on evaluation systems.

Tools:

-   LangSmith --- https://www.langchain.com/langsmith
-   Langfuse --- https://langfuse.com
-   Weights & Biases Weave --- https://wandb.ai/site/weave

------------------------------------------------------------------------

## Example: AI Support Agent for Your Hosting Company

Suppose you build an AI support assistant for a hosting company.

A beginner architecture:

``` text
Customer Question
↓
Prompt
↓
GPT
↓
Answer
```

A production architecture:

``` text
Customer Question
↓
Input Validation
↓
RAG Search
↓
Relevance Check
↓
Prompt Builder
↓
GPT
↓
JSON Output
↓
Schema Validation
↓
Safety Checks
↓
Fallback Retry
↓
Customer Response
↓
Logging & Evaluation
```

In this architecture, the prompt might be only 5% of the total
engineering effort.

------------------------------------------------------------------------

## Skills That Matter for an AI Engineer

For an AI Engineer role, the skills that usually matter most are:

1.  LLM APIs
2.  Structured Outputs
3.  RAG
4.  Evaluation Frameworks
5.  Guardrails & Safety
6.  AI System Design
7.  Cost Optimization
8.  Monitoring & Observability
9.  Fallback & Retry Strategies
10. Prompt Engineering

Prompt engineering is still important, but today it's often the smallest
component of a reliable production AI system. The real engineering
challenge is making the system dependable when the model behaves
unexpectedly.
