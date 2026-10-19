# From Hacky Script to “We Could Ship This”

This lab refactors a weekly meal-planner MVP into a more maintainable application. The project uses an LLM to generate a seven-day meal plan, validates the response with Pydantic, displays it through Gradio, and can send the result by email.

The main learning goal is to improve code quality without changing the user-facing behavior:

> Extract repeated logic, separate responsibilities, and make failures clear and actionable.

## Project path

Follow the lab in this order:

1. **Baseline** — open [`meal_planner_starter.ipynb`](meal_planner_starter.ipynb) and understand the original workflow.
2. **Refactoring notes** — use the sections below to identify repeated logic, large functions, and unclear errors.
3. **Refactored implementation** — open [`meal_planner_starter_refactored.ipynb`](meal_planner_starter_refactored.ipynb) and review the extracted helpers and error handling.
4. **Verification** — run the notebook’s test cases with valid and invalid inputs, then compare the result with the baseline.
5. **Submission** — record the changes, test evidence, and remaining risks in your lab submission.

## Repository structure

```text
.
├── README.md
├── instructions.md
├── meal_planner_starter.ipynb            # Original MVP / baseline
└── meal_planner_starter_refactored.ipynb # Refactored version
```

The notebooks follow the same application flow:

1. Installation and setup
2. Configuration
3. Pydantic models for validation
4. LangChain setup and prompts
5. Meal generation logic
6. Email functionality
7. Validation
8. Gradio interface
9. Application launch
10. Test cases

## Run the project

Open the refactored notebook in Jupyter or VS Code and run the cells from top to bottom.

Before running it, configure the credentials expected by the notebook. Keep secrets in a local `.env` file or environment variables; never commit API keys or email passwords.

Typical setup:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install jupyter notebook langchain langchain-openai langchain-anthropic pydantic gradio python-dotenv
```

The notebook reads `OPENAI_API_KEY` and `EMAIL_PASSWORD` from `.env`; update the sender address in its **Configuration** section if you want to test email delivery. Then run the **Launch Application** section to start the Gradio interface and the **Run Test Cases** section to verify the behavior.

## Refactoring checklist

### 1. Extract helper functions

Each repeated operation should live in one small, reusable function.

- `get_recipe_ingredients_string(recipe)` centralizes ingredient formatting.
- `parse_input_with_default(value, default)` handles empty or missing text-box values.
- `clean_llm_json(content)` removes Markdown code fences from model output before parsing.
- `EvaluationResult` standardizes evaluator output as a key, score, and comment.
- `format_error_response(error_msg)` keeps failure responses consistent.

### 2. Modularize responsibilities

Functions should have one clear job. For example:

| Component | Responsibility |
| --- | --- |
| `MealPlanGenerator.generate()` | Orchestrates generation and validation |
| Helper functions | Transform or clean data without side effects |
| `send_meal_plan_email()` | Sends the already-formatted email |
| `generate_plan_streaming()` | Adapts generation output for the Gradio UI |
| `send_email_ui()` | Validates UI input and reports status |
| Evaluators | Return a predictable evaluation result |

Keep pure transformations separate from side effects such as API calls, SMTP connections, and UI updates. This makes the logic easier to test without using live services or API tokens.

### 3. Improve error handling

An error should explain:

- what failed;
- where it failed;
- the error type or relevant context; and
- what the user should check next.

Exercise these scenarios:

| Scenario | Expected result |
| --- | --- |
| Missing or invalid OpenAI key | A credential-specific error and a configuration hint |
| Network failure | A connection error with a retry or connectivity suggestion |
| Malformed model JSON | A parsing error with location details when available |
| Invalid meal-plan schema | A validation error naming the violated constraint |
| Incorrect email credentials | A clear SMTP or authentication error |
| Empty UI input | The documented default value is applied |

Avoid catching every exception and returning a vague message. Handle expected failures specifically, preserve useful context, and keep the response shape predictable.

## Verification checklist

Before considering the refactor complete, confirm that:

- [ ] The valid-input flow still generates the expected meal plan.
- [ ] The plan contains the required number of recipes and ingredients.
- [ ] Helper functions work independently with small test inputs.
- [ ] Empty inputs use the intended defaults.
- [ ] Invalid JSON produces a useful parsing error.
- [ ] Schema violations identify the failed constraint.
- [ ] API, network, and email failures do not fail silently.
- [ ] Secrets are supplied through environment variables and are not committed.
- [ ] The refactored notebook is easier to follow than the baseline.

## Further improvements

After the core refactor is working, good next steps are:

- move reusable Python code out of the notebook into modules;
- add automated unit tests for pure helpers and validation;
- add structured logging instead of relying only on UI messages;
- introduce retry and timeout handling around external API calls;
- add a dependency file such as `requirements.txt` or `pyproject.toml`;
- add a small redacted example output so reviewers can understand the expected result without credentials.

See [`instructions.md`](instructions.md) for the complete lab brief and submission requirements.
