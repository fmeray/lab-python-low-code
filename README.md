# From Hacky Script to "We Could Ship This"


### Creating Helper Functions 
Goal: every piece of logic that appears more than once lives in one small function.

1. Collecting Ingredient Names from Every Recipe
    - This logic appears as inline list comprehensions in the Email Functionality section (format_meal_plan_email) and the Gardio Interface section (format_single_recipe)
    - Refactoring Solution:
        ```python
        def get_recipe_ingredients_string(recipe: dict) -> str:
            return ", ".join([f"{ing['quantity']} {ing['name']}" for ing in recipe['ingredients']])
2. If the textbox is empty, use a default.
    - This is written out manually twice inside the Gradio interface functions (generate_plan_streaming and send_email_ui)
    - Refactoring Solution:
        ```python
        def parse_input_with_default(value: str, default: str) -> str:
            return value if value and value.strip() else default
3. Removing Markdown Fences from Model Output
    - t is implemented once inside MealPlanGenerator.generate()
    - 
    ```python 
    def clean_llm_json(content: str) -> str:
    """Removes markdown code fences from LLM output string."""
        if "```json" in content:
            return content.split("```json")[1].split("```")[0].strip()
        elif "```" in content:
            return content.split("```")[1].split("```")[0].strip()
        return content.strip()
4. The Dictionary Each Evaluator Returns (key, score, comment)   
    ```python 
    class EvaluationResult(BaseModel):
        key: str
        score: float
        comment: str
5. Standardizing Error Message Formats
    - Error handling is fragmented across different formats
    - Refactoring Solution:
    ```python 
    def format_error_response(error_msg: str) -> dict:
        return {"success": False, "meal_plan": None, "error": str(error_msg)}
### Modularize the functions
Goal: each function does one job, and you can describe it in one sentence without using the word "and".
| Function Name | One-Sentence Description | Count of "and" | Modularity Note |
| --- | --- | --- | --- |
| **`generate()`** | Calls the LLM API, cleans the markdown JSON output, and handles validation errors.| **2** | High coupling; handles execution, cleaning, and error handling all in one place.|
| **`send_meal_plan_email()`** | Connects to the SMTP server to transmit the formatted meal plan text and handles delivery errors.| **1** | Manages network transport and exception handling combined.|
| **`generate_plan_streaming()`** | Calls the generator and yields text chunks sequentially to update the Gradio UI in real-time.| **1** | Acts as an adapter between the backend generator and the UI stream.|
| **`send_email_ui()`** | Validates the user's email address, triggers a fresh meal plan generation, and reports the status back to the interface.| **2** | Mixes input validation, business logic triggering, and UI feedback.|
| **`each evaluator`** | Evaluates model outputs against specific criteria to return a standardized score, comment, and metadata key. | **1** | Standardizes evaluation output structures. |

#### Function Modularization Architecture 
1. The Orchestrator Pattern
    - Before: Functions like generate() and send_email_ui() acted as monoliths, mixing string manipulation, API execution, error handling, and formatting all in one block.   
    - After: Core functions act purely as orchestrators. They read like a clean table of contents, delegating specific sub-tasks to dedicated helper functions and focusing solely on executing the sequence of steps in order.  
2. Pure Functions & Testability
    - Side effects (like live API calls or database connections) are strictly separated from data transformations (like JSON cleaning, input parsing, and Pydantic validation).
    - Benefit: You can test complex logic (e.g., markdown fence removal or schema validation) using mock inputs without needing to burn live API tokens or pay for network calls.
3. Layered Error Handling & Response Consistency
    - Errors are caught and formatted through standardized utility functions rather than scattered across raw exception blocks.
    - Responses adhere to predictable data dictionaries (success, meal_plan, error), making it trivial for downstream components (like Gradio UI streams or email dispatchers) to handle failures gracefully

### Adding Error Handling
Goal: Wen anything fails, the message names the function, the error type, the location, and a suggestion.
| Break this | How | What does it show? | After Refactoring
| --- | --- | --- | ---|
| OpenAI Key | Delete it, then try a fake one | Error: Error code: 401 - {'error': {'message': 'Incorrect API key provided: somethin************code. You can find your API key at https://platform.openai.com/account/api-keys.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_api_key'}} |OpenAIError: Missing credentials. Please pass an `api_key` |
| Network | Turn off Wi-fi, click Generate |"Could not parse server response: SyntaxError: Unexpected token '<'," |  Error: Function: unknown,[Error Type: OpenAIConnectionError, Location: General Execution , Suggestion: Connection error.|
|JSON from the model | Pass '{"recipes": [}' to the parsing code | Original Error Caught: {'success': False, 'meal_plan': None, 'error': 'JSON parsing error: Expecting value: line 1 column 14 (char 13)'}" | Function: parse_json_to_dict, Error Type: JSONDecodeError, Location: Line 1, Col 14 , Suggestion: LLM returned malformed JSON.' |
| Meal-plan schema | Feed a plan with 6 recipes or 4 ingredients |  Original Pydantic Validation Error Caught: 2 validation errors for MealPlan recipes, Value error, Meal plan must have exactly 7 recipes, got 0 - shopping_list Field required | Function: validate_and_build_meal_plan - Error Type: ValidationError - Location: Schema Constraints - Suggestion: LLM violated the required structure |
| Email | Use a wrong app password | Nothing | Failed to send email: Connection unexpectedly closed |
| Empty textbox | Leave Themes empty, then try None | Test Empty String ('') -> Result: 'random diverse cuisines,(Passed if default used). - Test None -> Result: 'random diverse cuisines' (Passed if default used) | |

### Test the refactored code
Goal: valid input behaves exactly like your Step 1 baseline, and every broken input from Step 5 now gives a clear message.