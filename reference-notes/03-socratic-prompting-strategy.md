# Socratic Prompting Strategy

## Problem with Direct Answers

A standard LLM prompt readily outputs direct answers, complete code snippets, or explicit step-by-step solutions. For interns, receiving immediate answers to every roadblock bypasses the productive struggle required for genuine learning and creates dependency on the AI.

## The KLI Framework

The Knowledge-Learning-Instruction (KLI) framework recommends "learner-instructed tutoring protocols" — the AI should guide the learner through discovery rather than acting as a task-completion engine.

## Prompt Design Principles

### 1. Socratic Method Enforcement

The system prompt must explicitly forbid providing the final command, raw output, or direct solution. Instead, ask guiding questions that lead the intern to the answer.

**Bad** (direct answer):
> "Click the blue deploy button in the top right corner."

**Good** (Socratic guidance):
> "I see you've configured the environment variables correctly. Based on the deployment checklist, what's the final validation step before initialization?"

### 2. Few-Shot Examples in the System Prompt

Include in-context examples of ideal pedagogical interactions so the model learns the expected tutoring pattern:

```
example interaction:
- intern (looking at a git merge conflict): "i don't know what to do here"
- you: "okay so you've got a merge conflict in this file. you can see the 
  markers showing your changes versus the incoming changes. which version 
  has the code you actually want to keep? look at both sections and tell 
  me which one looks right to you."
```

### 3. Adaptive Scaffolding via Conversation History

Because the app maintains a 10-exchange conversation history, the LLM can adapt its scaffolding level dynamically:

- **Intern struggling**: Increase scaffolding — more specific hints, more pointing
- **Intern demonstrating understanding**: Fade assistance — brief affirmative validation, less hand-holding
- **Intern answering Socratic questions correctly**: Transition to "what would you try next?" posture

### 4. Proactive Intervention Tone

When Tutor Mode triggers (idle detection), the prompt should enforce a low-pressure, curious tone:

```
"hey, looks like you might be thinking through something. i can see 
[specific thing on screen]. have you tried [relevant next step]?"
```

Never:
```
"You've been idle for 60 seconds. Here's what you should do..."
```

### 5. Actionable Endings (Not Yes/No Questions)

Instead of dead-end closers like "want me to explain more?", end with:
- A next step the intern should take
- A common mistake to watch out for
- A related concept that connects to what they're doing

## When Direct Answers ARE Appropriate

The Socratic approach should yield to directness when:
- The intern explicitly asks for the answer ("just tell me the command")
- It's a factual lookup, not a learning moment ("what port does postgres use?")
- Safety is involved ("am I about to delete production data?")
- The intern has already demonstrated understanding and just needs confirmation
