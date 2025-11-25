# How the ARC-AGI Solver Works

This document provides a comprehensive explanation of the Poetiq ARC-AGI solver's architecture and functionality.

## High-Level Overview

The solver is designed to solve Abstract Reasoning Corpus (ARC) tasks by leveraging the power of large language models (LLMs) to generate Python code. It employs a multi-expert, iterative, and feedback-driven approach to guide the LLM towards a correct solution.

The core idea is to treat the ARC problem as a code-generation task. The solver asks the LLM to write a Python function that transforms the input grid into the output grid. The generated code is then executed in a sandbox, and its output is compared to the expected output. Based on the result, the solver provides feedback to the LLM, which then refines its code in the next iteration.

This process is repeated until a correct solution is found or a maximum number of iterations is reached. To increase the chances of success, the solver runs multiple "experts" in parallel, each with a slightly different configuration. The results from these experts are then aggregated and ranked to produce the final answer.

### Main Components

The solver is composed of the following key components:

*   **Configuration (`config.py`):** Defines the parameters for the LLM, the solver, and the voting mechanism. This allows for easy experimentation with different settings.
*   **Parallel Solving (`solve_parallel_coding.py`):** Manages the parallel execution of multiple solver instances (experts).
*   **Core Solving Loop (`solve_coding.py`):** The heart of the solver, responsible for interacting with the LLM, executing the generated code, and providing feedback.
*   **Prompts (`prompts.py`):** Contains the templates for the prompts sent to the LLM, which are crucial for guiding it towards a correct solution.
*   **Sandboxing (`sandbox.py`):** Provides a secure environment for executing the LLM-generated code.
*   **LLM Interaction (`llm.py`):** Handles the communication with the LLM API.
*   **Data Structures (`types.py`):** Defines the data structures used throughout the codebase.
*   **Input/Output (`io.py`):** Handles the loading of the ARC problems.
*   **Scoring (`scoring.py`):** Provides functions for evaluating the correctness of a solution.
*   **Utilities (`utils.py`):** Contains various helper functions.

## Detailed Configuration (`config.py`)

The `config.py` file is the central hub for configuring the solver. It defines a list of `ExpertConfig` dictionaries, each specifying the parameters for a single "expert." The `CONFIG_LIST` variable holds these configurations. By default, it's configured to run a single expert, but can be easily scaled up to run multiple experts by changing the `NUM_EXPERTS` variable.

Each `ExpertConfig` dictionary contains the following key parameters:

### Prompts

*   **`solver_prompt`:** The main prompt used to instruct the LLM on how to solve the ARC task. There are several versions of this prompt available in `prompts.py` (`SOLVER_PROMPT_1`, `SOLVER_PROMPT_2`, etc.), each with a slightly different emphasis.
*   **`feedback_prompt`:** The prompt used to provide feedback to the LLM on its previous incorrect solutions. This prompt is appended to the `solver_prompt` in subsequent iterations.

### LLM Parameters

*   **`llm_id`:** The identifier for the language model to be used (e.g., `'gemini/gemini-3-pro-preview'`).
*   **`solver_temperature`:** The temperature setting for the LLM, which controls the randomness of the output. A higher temperature (e.g., 1.0) results in more creative and diverse responses, while a lower temperature results in more deterministic and focused responses.
*   **`request_timeout`:** The timeout for a single request to the LLM, in seconds.
*   **`max_total_timeouts`:** The maximum number of timeouts allowed per problem per solver.
*   **`max_total_time`:** The maximum total time allowed per problem per solver.
*   **`per_iteration_retries`:** The number of times to retry a request to the LLM if it fails.

### Solver Parameters

*   **`num_experts`:** The total number of experts to run in parallel.
*   **`max_iterations`:** The maximum number of iterations to run the solver for a single problem.
*   **`max_solutions`:** The maximum number of previous solutions to include in the feedback prompt.
*   **`selection_probability`:** The probability of selecting a previous solution to include in the feedback prompt.
*   **`seed`:** The random seed for the solver, used for reproducibility.
*   **`shuffle_examples`:** Whether to shuffle the training examples in the prompt.
*   **`improving_order`:** Whether to present the previous solutions in the feedback prompt in order of improving score.
*   **`return_best_result`:** Whether to return the best result found so far if no correct solution is found within the maximum number of iterations.

### Voting Parameters

*   **`use_new_voting`:** A boolean that determines whether to use the new or old voting system. The new system is more sophisticated and uses a diversity-first approach.
*   **`count_failed_matches`:** If true, the voting system will consider failed solutions that produce the same output as a successful solution.
*   **`iters_tiebreak`:** A boolean that determines whether to use the number of iterations as a tiebreaker when ranking solutions.
*   **`low_to_high_iters`:** If `iters_tiebreak` is true, this determines whether to prioritize solutions with fewer or more iterations.

## Parallel Solving (`solve_parallel_coding.py`)

To improve the chances of finding a correct solution and to explore a wider range of potential solutions, the solver employs a parallel solving mechanism. This is managed by the `solve_parallel_coding` function in `solve_parallel_coding.py`.

The basic idea is to run multiple instances of the `solve_coding` function concurrently, each with its own expert configuration. This is achieved using Python's `asyncio` library, which allows for efficient handling of asynchronous tasks.

Here's a step-by-step breakdown of the process:

1.  **Initialization:** The `solve_parallel_coding` function receives the training and test data, as well as a list of expert configurations. It then initializes a separate random seed for each expert to ensure that they explore different solution spaces.

2.  **Concurrent Execution:** The function creates an `asyncio` task for each expert configuration, which calls the `solve_coding` function. The `asyncio.gather` function is then used to run all the tasks concurrently.

3.  **Result Aggregation:** Once all the tasks are complete, the results from each expert are collected into a list of `ARCAGIResult` objects.

4.  **Ranking and Voting:** The results are then passed to a ranking and voting mechanism, which is described in more detail in a later section.

By running multiple experts in parallel, the solver can explore different prompting strategies, LLM temperatures, and other parameters simultaneously. This significantly increases the likelihood of finding a correct solution in a timely manner.

## Core Solving Loop (`solve_coding.py`)

The `solve_coding.py` file contains the core logic of the solver. The `solve_coding` function implements an iterative process that guides the LLM towards a correct solution. Here's a detailed breakdown of the steps involved in each iteration:

### 1. Prompt Engineering

The first step is to construct a high-quality prompt that will be sent to the LLM. This is a crucial step, as the quality of the prompt directly influences the quality of the generated code.

*   **Initial Prompt:** In the first iteration, the prompt consists of the `solver_prompt` from the expert's configuration, with the `$$problem$$` placeholder replaced by a textual representation of the ARC problem. The `solver_prompt` provides the LLM with instructions on how to approach the problem, including examples of good solutions.

*   **Feedback-Driven Prompt:** In subsequent iterations, the prompt is augmented with a `feedback_prompt`. This prompt includes the code from previous incorrect solutions, along with feedback on why they were incorrect. This feedback helps the LLM to learn from its mistakes and avoid repeating them. The `create_examples` function is used to format this feedback, and the `selection_probability` parameter controls which of the previous solutions are included.

### 2. LLM Interaction

Once the prompt is constructed, it is sent to the LLM using the `llm` function in `llm.py`. This function handles the API call to the specified language model (e.g., Gemini) and returns the generated code.

### 3. Code Parsing

The response from the LLM is a markdown-formatted string that contains a Python code block. The `_parse_code_from_llm` function is used to extract the Python code from this string.

### 4. Sandboxed Execution

To ensure safety and prevent the LLM-generated code from executing arbitrary commands, the code is run in a secure sandbox. The `run` function in `sandbox.py` is responsible for this. It executes the code in a separate process with restricted permissions and a timeout.

### 5. Evaluation

The output of the sandboxed execution is then evaluated to determine its correctness. The `_eval_on_train_and_test` function performs this evaluation.

*   **Training Set:** For each training example, the output of the code is compared to the expected output. A `RunResult` object is created for each example, which contains a boolean `success` flag, the raw output string, a "soft score" (a measure of similarity to the expected output), and any errors that occurred.

*   **Test Set:** The code is also run on the test inputs, and `RunResult` objects are created for these as well.

### 6. Feedback Generation

If the generated code does not solve all the training examples correctly, feedback is generated to guide the LLM in the next iteration. The `_build_feedback` function is responsible for this. It creates a detailed feedback string that explains which examples were solved incorrectly and why. This feedback includes a diff grid that visually highlights the differences between the predicted output and the expected output.

### 7. Iteration

The process then repeats from step 1, with the generated feedback incorporated into the prompt for the next iteration. This iterative refinement continues until a correct solution is found or the `max_iterations` limit is reached.

## Result Aggregation and Ranking

After all the parallel experts have completed their execution, their results need to be aggregated and ranked to produce the final answer. This is handled by the `solve_parallel_coding` function.

The ranking process is designed to prioritize solutions that are both correct and have been independently discovered by multiple experts. It also aims to promote diversity in the final ranking.

Here's how it works:

1.  **Bucketing:** The results are first grouped into "buckets" based on their output on the test data. The `canonical_test_key` function is used to create a unique identifier for each distinct test output. This means that solutions that produce the same output on the test set will be placed in the same bucket.

2.  **Separating Passers and Failers:** The buckets are then separated into two groups: "passers" (solutions that correctly solved all the training examples) and "failers" (solutions that did not).

3.  **Ranking Passers:** The passer buckets are ranked primarily by the number of "votes" they received (i.e., the number of experts in the bucket). The more experts that independently came up with the same solution, the higher its rank.

4.  **Ranking Failers:** The failer buckets are also ranked by the number of votes, but they are also ranked by their mean "soft score." This allows the solver to identify promising but incorrect solutions.

5.  **Diversity-First Ordering:** The final ranked list of solutions is constructed using a diversity-first approach. The top-ranked solution from each passer bucket is added to the list first, followed by the top-ranked solution from each failer bucket. This ensures that a diverse range of solutions is presented, even if some have fewer votes.

6.  **Final List:** The remaining solutions from each bucket are then appended to the list, resulting in a comprehensive and well-ranked list of all the solutions found by the experts.

This sophisticated ranking and voting mechanism is a key feature of the solver, as it allows for a robust and reliable way to combine the results from multiple experts and select the most promising solution.

## Key Strengths of the Approach

The Poetiq ARC-AGI solver's architecture and methodology offer several key advantages:

*   **Multi-Expert Approach:** By running multiple experts with different configurations in parallel, the solver can explore a wider range of potential solutions and is more resilient to getting stuck in a local optimum. This diversity of approaches significantly increases the chances of finding a correct solution.

*   **Iterative Refinement:** The iterative nature of the core solving loop allows the LLM to learn from its mistakes. The feedback-driven prompt engineering provides targeted guidance, helping the LLM to progressively improve its code until it finds a correct solution.

*   **Robust Evaluation:** The use of a secure sandbox for code execution and a comprehensive evaluation process ensures that the generated solutions are both safe and correct. The "soft score" metric provides a nuanced measure of similarity, which is useful for guiding the search for a solution.

*   **Sophisticated Ranking:** The diversity-first ranking and voting mechanism is a powerful tool for aggregating the results from multiple experts. It prioritizes consensus and correctness while also promoting a diverse range of solutions.

*   **Flexibility and Extensibility:** The solver's modular design and configurable architecture make it easy to experiment with different LLMs, prompts, and solving strategies. This flexibility is crucial for adapting to the evolving landscape of AI and for tackling new and more challenging reasoning tasks.

In conclusion, the Poetiq ARC-AGI solver is a powerful and sophisticated system that combines the strengths of large language models with a robust and intelligent search and evaluation framework. Its multi-expert, iterative, and feedback-driven approach represents a significant step forward in the field of automated reasoning.
