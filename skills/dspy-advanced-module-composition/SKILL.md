---
name: dspy-advanced-module-composition
version: "1.0.0"
dspy-compatibility: "2.5+"
description: This skill should be used when the user asks to "compose DSPy modules", "use dspy.Ensemble", "combine multiple modules", "use dspy.MultiChainComparison", mentions "ensemble voting", "module composition", "parallel execution", "sequential pipelines", or needs to build complex multi-module DSPy programs with ensemble patterns or multi-chain comparison.
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
---

# DSPy Advanced Module Composition

## Goal

Compose complex DSPy programs using Ensemble voting, MultiChainComparison, and sequential/parallel module patterns.

## When to Use

- Need consensus from multiple approaches
- Comparing different reasoning strategies
- Building robust pipelines with fallbacks
- Complex multi-step workflows with branching
- Ensemble methods for improved accuracy

## Related Skills

- Design modules: [dspy-custom-module-design](../dspy-custom-module-design/SKILL.md)
- Define signatures: [dspy-signature-designer](../dspy-signature-designer/SKILL.md)
- Evaluate performance: [dspy-evaluation-suite](../dspy-evaluation-suite/SKILL.md)

## Inputs

| Input | Type | Description |
|-------|------|-------------|
| `modules` | `list[dspy.Module]` | Modules to compose |
| `composition_type` | `str` | "ensemble", "sequential", "comparison" |

## Outputs

| Output | Type | Description |
|--------|------|-------------|
| `composed_program` | `dspy.Module` | Composed multi-module program |

## Workflow

### Phase 1: Ensemble Voting

Combine multiple modules and vote on outputs:

```python
import dspy

dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

# Create multiple approaches
qa1 = dspy.Predict("question -> answer")
qa2 = dspy.ChainOfThought("question -> answer")
qa3 = dspy.ReAct("question -> answer", tools=[])

# Ensemble: run all, vote on best
ensemble = dspy.Ensemble(
    modules=[qa1, qa2, qa3],
    strategy="majority_vote"  # or "average" for numeric outputs
)

result = ensemble(question="What is 2 + 2?")
print(result.answer)  # Voted answer
```

### Phase 2: MultiChainComparison

Compare reasoning approaches explicitly:

```python
import dspy

class ComparisonPipeline(dspy.Module):
    def __init__(self):
        # Different reasoning strategies
        self.direct = dspy.Predict("question -> answer, reasoning")
        self.cot = dspy.ChainOfThought("question -> answer")
        self.react = dspy.ReAct("question -> answer", tools=[self.search])

        # Compare and select best
        self.compare = dspy.MultiChainComparison(
            modules=[self.direct, self.cot, self.react],
            comparison_metric="confidence"  # or custom function
        )

    def search(self, query: str) -> str:
        """Dummy search tool."""
        return "Search results for: " + query

    def forward(self, question):
        return self.compare(question=question)

# Usage
pipeline = ComparisonPipeline()
result = pipeline(question="Explain quantum computing")
print(f"Best answer: {result.answer}")
print(f"Strategy used: {result.selected_module}")
```

### Phase 3: Sequential Composition

Chain modules for multi-step workflows:

```python
import dspy

class SequentialRAG(dspy.Module):
    """Multi-step RAG pipeline."""

    def __init__(self):
        # Step 1: Query rewriting
        self.rewrite = dspy.Predict("question -> refined_query: str")

        # Step 2: Retrieval
        self.retrieve = dspy.Retrieve(k=5)

        # Step 3: Answer generation
        self.generate = dspy.ChainOfThought("context, question -> answer")

        # Step 4: Validation
        self.validate = dspy.Predict("answer, question -> is_valid: bool, confidence: float")

    def forward(self, question):
        # Sequential execution
        refined = self.rewrite(question=question)
        passages = self.retrieve(refined.refined_query).passages

        answer_pred = self.generate(
            context=passages,
            question=question
        )

        validation = self.validate(
            answer=answer_pred.answer,
            question=question
        )

        return dspy.Prediction(
            answer=answer_pred.answer,
            is_valid=validation.is_valid,
            confidence=validation.confidence
        )

# Usage
rag = SequentialRAG()
result = rag(question="What causes lightning?")
print(f"Answer: {result.answer} (valid: {result.is_valid})")
```

### Phase 4: Fallback Strategies

Handle failures with fallback modules:

```python
import dspy
import logging

logger = logging.getLogger(__name__)

class RobustQA(dspy.Module):
    """Fallback strategy for errors."""

    def __init__(self):
        self.primary = dspy.ChainOfThought("question -> answer")
        self.fallback = dspy.Predict("question -> answer")

    def forward(self, question):
        try:
            result = self.primary(question=question)
            if result.answer and len(result.answer) > 10:
                return result
        except Exception as e:
            logger.error(f"Primary failed: {e}")

        return self.fallback(question=question)
```

## Production Example

```python
import dspy
from typing import List

class MultiStrategyQA(dspy.Module):
    """Production QA with ensemble."""

    def __init__(self):
        self.retrieve = dspy.Retrieve(k=3)
        self.fast = dspy.Predict("context, question -> answer")
        self.thorough = dspy.ChainOfThought("context, question -> answer")
        self.ensemble = dspy.Ensemble(
            modules=[self.fast, self.thorough],
            strategy="majority_vote"
        )

    def forward(self, question: str):
        context = self.retrieve(question).passages
        return self.ensemble(context=context, question=question)

# Usage with optimization
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))
qa = MultiStrategyQA(retriever_url="http://...")

# Optimize ensemble
from dspy.teleprompt import BootstrapFewShot

optimizer = BootstrapFewShot(
    metric=lambda ex, pred, trace: ex.answer in pred.answer,
    max_bootstrapped_demos=3
)

compiled = optimizer.compile(qa, trainset=trainset)
```

## Best Practices

1. **Test modules independently** - Validate each module before composition
2. **Handle failures gracefully** - Use try/except in parallel composition
3. **Balance cost vs accuracy** - Ensembles are expensive (N × cost)
4. **Optimize composed programs** - Use BootstrapFewShot or MIPROv2 on final composition
5. **Module reusability** - Design modules to work in multiple compositions

## Limitations

- Ensemble increases cost linearly with module count
- Voting strategies may not work for all output types
- Sequential composition amplifies latency
- Error propagation in chains can be hard to debug
- Parallel composition requires careful state management
