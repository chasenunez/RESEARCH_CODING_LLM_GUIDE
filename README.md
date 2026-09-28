*A quick forward for the humans in the room:* This is mostly a collection of ideas. It should not be viewed as authoratative, nor accepted without critical thinking. This repo is open for additions, subtractions, ammendments, and disagreement. Please feel free to make a productive PR with your thoughts. 


# Using Large Language Models for Research Coding: A Practical Guide

Large language models are, by design, not compatible with reproducible coding. They are non-deterministic, their internal states are opaque, their training data is undisclosed, and their outputs cannot be guaranteed to be stable across time, versions, or even consecutive identical requests. No amount of parameter tuning changes this fundamental characteristic.

LLMs generate text that is statistically plausible, not verified. This distinction is critical for research. They can produce code that *looks* correct and runs without errors but yields wrong results. They routinely fabricate function signatures, package names, and API parameters that do not exist — a phenomenon widely known as "hallucination." They have no access to your runtime environment and cannot know which versions of libraries you have installed. They do not understand your scientific intent: an LLM optimises for token-level plausibility, not for the correctness of your analysis. They present all output with the same tone of authority, regardless of whether the answer is well-established or completely fabricated — and if you ask an LLM whether its own code is correct, it will almost always say yes. Commercial LLM APIs do not guarantee deterministic output: even with a temperature of 0, small variations can occur between runs due to hardware, load-balancing, and silent model updates.

But LLMs are not responsible for reproducible code. **You are.**

An LLM is a tool — like a calculator, a compiler, or a colleague's suggestion scribbled on a whiteboard. The reproducibility of your research does not depend on whether you used an LLM. It depends on what you did with the output: whether you verified it, documented it, tested it, and made it transparent. Researchers have always relied on tools they did not fully control. What matters is the discipline you bring to the process.

This also means you should consider whether an LLM is the right tool for a given task. If you already know how to write the function you need, writing it yourself may be faster and more reliable than prompting, reviewing, and correcting LLM output. For critical numerical algorithms where every line must be scientifically justified, the overhead of verifying LLM-generated code may exceed the cost of writing it from scratch. LLMs are most useful for boilerplate, scaffolding, unfamiliar syntax, and exploratory drafting — tasks where speed matters and the output is easy to verify.

This guide shows you how to use LLMs to assist your coding work while maintaining — and in some cases strengthening — the reproducibility, transparency, and reusability of your research software. It is organised around the three phases of the workflow: preparing to prompt, working with generated code, and preparing to publish.

The core approach is simple: **prompt small, verify everything, document thoroughly.**

---

## Phase 1: Before You Prompt

### Decompose the Task

The single most effective habit for getting reliable, reproducible code from an LLM is to **never ask it to write an entire program at once.** Large, open-ended prompts ("create a program that analyses my hydrology data and produces a report") give the model maximum room to hallucinate, make silent assumptions, invent dependencies, and produce monolithic code that is difficult to test, understand, or reuse.

Instead, decompose your task into individual functions, and prompt for each one separately with precise specifications. This mirrors good software engineering practice — and it happens to be exactly the kind of constraint that makes LLM output more reliable.

### One Function, One Prompt

For each function you need, your prompt should specify four things explicitly:

1. **What goes in** — the exact input types, shapes, and units
2. **What happens** — the specific operation or algorithm, ideally with a reference
3. **What comes out** — the exact output type, shape, and meaning
4. **What can go wrong** — the edge cases and how to handle them

Rather than asking "write me a function to do X," supply the specific library versions you are using, the data structures involved, and an example of the expected input and output. The more specific your prompt, the less room the model has to invent. Ask the model to explain its approach before writing code — this makes errors in logic visible and gives you an opportunity to correct course before code is generated.

### Bad Prompt vs. Good Prompt

**Bad — vague, open-ended, invites hallucination:**

> "Write a Python program that reads my CSV files, cleans the data, does some statistical analysis, and makes nice plots."

This prompt will produce a long script full of assumptions about your file structure, column names, cleaning logic, statistical methods, and plot aesthetics. Many of those assumptions will need correction, and untangling them from a monolithic script is slow, frustrating work.

**Good — specific, constrained, one function at a time:**

> "I need a Python function called `load_sensor_data` that takes a single argument `filepath` (a string pointing to a CSV file). The CSV has columns 'timestamp' (ISO 8601 format), 'temperature_c' (float, degrees Celsius), and 'conductivity_us' (float, microsiemens/cm). The function should parse the timestamp column into pandas datetime, drop any rows where temperature or conductivity is NaN, and return a pandas DataFrame with those three columns. Use only pandas (version 2.2.x). Raise a FileNotFoundError if the file does not exist and a ValueError if expected columns are missing."

This prompt constrains the model tightly. The output will be short, testable, and easy to verify against a known input file.

### Set Model Parameters for Reproducibility

When accessing an LLM via an API, set the temperature parameter as low as possible (0.0–0.2) for code-generation tasks. Low temperature makes the model select the most probable tokens, reducing creative but incorrect output. For code generation specifically, avoid applying frequency or presence penalties, since code naturally and legitimately repeats variable names, keywords, and structural patterns. If the API supports a seed parameter, set one and record it.

If your research involves LLM-generated or LLM-assisted code, you must document the conditions under which it was generated so that others can evaluate and, where possible, reproduce your work. Current guidelines for empirical studies involving LLMs — including those developed by Baltes et al. (2025) and adopted at venues such as ICSE — recommend reporting the following:

**Model identity and version.** Record the exact model name and version string (e.g., `gpt-4-0125-preview`, `claude-sonnet-4-20250514`). For locally hosted models, record the Hugging Face model card identifier, the quantisation format, and the inference framework (e.g., Ollama, vLLM, llama.cpp) and its version.

**All sampling parameters.** At minimum: temperature, top_p, top_k (if applicable), max_tokens, any penalty parameters, and the seed value if one was set. If you used default settings, state that explicitly — defaults change between versions.

**The system fingerprint or equivalent.** Some providers (e.g., OpenAI) return a `system_fingerprint` with each API response. Record it. It helps identify when the underlying infrastructure has changed.

**The date of each query.** Commercial models can be updated without notice. The date of your interaction is part of the provenance chain.

**The full prompt and the full response.** Save the complete prompt text, including any system prompt. If you used a multi-turn conversation, save the entire conversation log. Save the complete model output, not just the code you chose to use. Prompts are part of your experimental method; the full response allows others to see what was generated versus what you modified.

### Log Your Interactions Automatically

Create a simple logging wrapper around your LLM API calls that automatically captures these parameters and saves them to a structured file (JSON or YAML). An example pattern:

```python
# Minimal logging of an LLM API call for reproducibility.
# Adapt to your provider's SDK as needed.

import json
import datetime

def log_llm_call(prompt, response, model, params, output_path):
    """
    Save a complete record of an LLM interaction for reproducibility.

    Args:
        prompt: The full prompt text sent to the model.
        response: The complete response object from the API.
        model: Model identifier string (e.g., 'gpt-4-0125-preview').
        params: Dict of sampling parameters (temperature, top_p, seed, etc.).
        output_path: Path to the JSON log file.
    """
    record = {
        "timestamp": datetime.datetime.utcnow().isoformat() + "Z",
        "model": model,
        "parameters": params,
        "prompt": prompt,
        "response_text": response,  # or response.text, depending on SDK
        "system_fingerprint": None,  # populate if available from provider
    }
    with open(output_path, "a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

Store these logs alongside your code in version control. They are part of your research record.

---

## Phase 2: While You Work

### A Worked Example: Building an Analysis Step by Step

Suppose you need to process sensor data from a field station: load it, remove outliers, calculate daily means, and plot the result. Instead of one prompt, use four:

**Prompt 1 — Load data:**
> "Write a Python function `load_sensor_data(filepath: str) -> pd.DataFrame` that reads a CSV with columns 'timestamp', 'temperature_c', 'conductivity_us'. Parse timestamps as datetime, drop rows with NaN in either measurement column. Return the cleaned DataFrame. Use pandas 2.2.x only. Raise FileNotFoundError if path is invalid, ValueError if columns are missing."

**Prompt 2 — Remove outliers:**
> "Write a Python function `remove_outliers(df: pd.DataFrame, column: str, n_sigma: float = 3.0) -> pd.DataFrame` that removes rows where the value in `column` is more than `n_sigma` standard deviations from the mean. Return a new DataFrame (do not modify the input). Raise KeyError if the column does not exist. Use only pandas and numpy."

**Prompt 3 — Calculate daily means:**
> "Write a Python function `daily_means(df: pd.DataFrame, timestamp_col: str = 'timestamp', value_cols: list[str] = None) -> pd.DataFrame` that groups by calendar date (derived from the timestamp column) and returns the mean of each value column. If value_cols is None, use all numeric columns. The returned DataFrame should have 'date' as the index. Use pandas only."

**Prompt 4 — Plot the result:**
> "Write a Python function `plot_timeseries(df: pd.DataFrame, y_columns: list[str], title: str, output_path: str) -> None` that creates a line plot with date on the x-axis and one line per column in y_columns. Use matplotlib 3.9.x. Label axes with column names. Save the figure to output_path at 300 DPI. Do not call plt.show()."

Each of these functions can be independently tested, reviewed, and replaced. If the outlier function is wrong, you fix one function — not an entire script.

### Verify Each Function Before Moving On

Once the LLM returns a function, before moving to the next prompt:

1. **Read the code yourself.** Do you understand every line? If not, ask the LLM to explain the parts you don't understand — but verify its explanation against official documentation, not by asking the same LLM to confirm itself.
2. **Check the imports.** Does the function use only the libraries you specified? Are the function names and signatures real? Verify against the official docs for the library and version in question.
3. **Write a quick test.** Create a small test input where you know the expected output, and run the function against it. For numerical or analytical code, test against a case with a known correct answer.
4. **Run static analysis.** Pass the code through a linter and type checker (mypy, pylint, ESLint, etc.) to catch issues the LLM may have introduced.

### Code Structure: What LLMs Get Wrong

Whether code is human-written or LLM-assisted, the same principles of good software engineering apply. But LLMs violate certain principles so reliably that it is worth watching for them explicitly.

**Monolithic functions.** LLMs tend to produce single large functions that read data, process it, and plot results in one block. Refactor into separate functions, each with a single responsibility.

**Magic numbers.** LLMs frequently embed specific values — thresholds, file paths, column indices — directly in code rather than making them configurable. Replace hard-coded values with named constants or configuration parameters.

**Missing dependency specifications.** Every external library your code uses should be listed in a dependency file (requirements.txt, environment.yml, renv.lock, etc.) with pinned version numbers. LLMs almost never generate complete or correct dependency specifications on their own — you must do this yourself.

**No error handling.** LLMs tend to generate code that assumes all inputs are well-formed. Add input validation and informative error messages.

**Unnecessary cleverness.** LLMs sometimes generate overly clever code — dense list comprehensions, obscure built-in tricks, premature optimisations. Prefer clarity over cleverness. A research colleague reading your code in three years should be able to understand it without consulting the LLM that wrote it.

### Assembling Functions into a Pipeline

Once you have a set of tested, documented functions, write a short main script or pipeline that calls them in sequence. This top-level script is the part *you* write — it encodes your scientific workflow and decision-making. The LLM helped with the building blocks; you are the architect.

```python
"""
Pipeline: daily mean analysis of field sensor data.
Author: [Your Name]
Date: 2026-04-29
Note: Individual functions were drafted with LLM assistance (see provenance
      comments in each module). The pipeline logic and parameter choices
      are the author's own.
"""
from sensor_io import load_sensor_data
from cleaning import remove_outliers
from aggregation import daily_means
from plotting import plot_timeseries

# --- Configuration (all tuneable parameters in one place) ---
INPUT_FILE = "data/station_alpha_2025.csv"
OUTLIER_THRESHOLD = 3.0  # standard deviations; see Methods section of paper
VALUE_COLUMNS = ["temperature_c", "conductivity_us"]
OUTPUT_PLOT = "figures/daily_means.png"

# --- Execution ---
df = load_sensor_data(INPUT_FILE)
df = remove_outliers(df, column="temperature_c", n_sigma=OUTLIER_THRESHOLD)
df = remove_outliers(df, column="conductivity_us", n_sigma=OUTLIER_THRESHOLD)
daily = daily_means(df, value_cols=VALUE_COLUMNS)
plot_timeseries(daily, y_columns=VALUE_COLUMNS,
                title="Daily Mean Sensor Readings — Station Alpha",
                output_path=OUTPUT_PLOT)
```

This pipeline is readable, auditable, and FAIR-friendly: every parameter is visible, every step is a named function that can be inspected independently, and the configuration is separated from the logic.

### Commenting and Documenting LLM-Assisted Code

Code comments serve a different purpose in LLM-assisted work than in fully human-written code. They must communicate not only *what* the code does but also *where it came from* and *how much you trust it*.

**Provenance comments.** At the top of any file or function that was substantially generated or modified by an LLM, add a provenance comment:

```python
# ----- Provenance -----
# This function was initially generated by Claude (claude-sonnet-4-20250514)
# on 2026-04-15 in response to the prompt stored in ./llm_logs/sort_func.json.
# It was subsequently reviewed, tested, and modified by [Your Name].
# Modifications: added input validation, fixed off-by-one error in line 34,
# replaced fabricated 'np.fastsort' call with np.sort.
# -----------------------
```

This serves three purposes: it satisfies transparency requirements from publishers and funders (e.g., the ACM policy on authorship requires disclosure of generative AI usage), it helps future readers (including your future self) understand the code's origins, and it flags which parts may need extra scrutiny.

**Inline comments: explain *why*, not *what*.** LLMs tend to generate comments that restate the code in English ("increment counter by 1") rather than explaining the reasoning. Replace these with comments that explain intent, assumptions, and scientific context:

```python
# WRONG — restates the code:
x = x + 1  # increment x by one

# RIGHT — explains the reasoning:
x = x + 1  # shift to 1-based indexing to match Table 2 in Mueller et al. (2024)
```

**Docstrings.** Every function should have a docstring that describes its purpose, parameters, return value, exceptions, and an example of usage if non-trivial. Use a consistent format (e.g., NumPy-style or Google-style for Python). LLMs can draft docstrings, but you must verify that the parameter descriptions match the actual behaviour and that stated assumptions are correct.

```python
def calculate_flux(concentration, velocity, cross_section):
    """
    Calculate mass flux through a cross-section.

    Implements Eq. 3 from Müller et al. (2024), J. Hydrology, 612, 128.
    Assumes steady-state conditions and uniform velocity profile.

    Args:
        concentration (float): Solute concentration in mg/L.
        velocity (float): Flow velocity in m/s. Must be positive.
        cross_section (float): Cross-sectional area in m². Must be positive.

    Returns:
        float: Mass flux in mg/s.

    Raises:
        ValueError: If velocity or cross_section is non-positive.

    Example:
        >>> calculate_flux(5.0, 0.3, 2.0)
        3.0
    """
    if velocity <= 0 or cross_section <= 0:
        raise ValueError("velocity and cross_section must be positive")
    return concentration * velocity * cross_section
```

### Summary of the Function-by-Function Approach

| Instead of this... | Do this... |
|---|---|
| "Write a program that does X" | "Write a function that takes Y, does Z, returns Q" |
| One long prompt for the whole task | One prompt per function, with explicit I/O specs |
| Accepting a 200-line script as-is | Testing each function independently before assembly |
| Letting the LLM choose libraries | Specifying the exact library and version in the prompt |
| Asking "is this correct?" | Testing against known inputs and expected outputs |
| Trusting LLM-generated comments | Rewriting comments to explain *why*, not *what* |

---

## Phase 3: Before You Publish

### Version Control

Use Git (or another version control system) throughout your project, not just at the end. Version control is arguably more important to reproducibility than any LLM parameter setting. It gives you a complete history of how your code evolved, makes it easy to revert mistakes, and provides the natural home for the LLM interaction logs and provenance comments described above. Commit early and often, with clear commit messages that distinguish between your own work and LLM-assisted additions.

### Transparency and Disclosure

Most publishers and funding bodies now require disclosure of generative AI use. The ACM, for example, mandates that any use of generative AI to create content must be declared. Even where not formally required, transparency is good scientific practice.

In your publication or supplementary materials, state that an LLM was used and in what capacity (e.g., code generation, debugging, documentation), which model and version was used, and that you reviewed, tested, and take responsibility for the output.

In your code repository, include the LLM interaction logs (prompts and responses) in a dedicated directory, annotate LLM-generated or LLM-modified code with provenance comments, and describe your verification process in the README or a CONTRIBUTING file.

### Responsibility

Using an LLM does not transfer responsibility for your code. You are the author. If the LLM introduced a bug, an incorrect assumption, or a fabricated dependency, the responsibility lies with you. This is both an ethical and practical reality: reviewers, collaborators, and users of your software will hold you accountable for what it does, regardless of how it was written.

---

## Quick-Reference Checklist

Use this checklist when incorporating LLM-generated code into a research project.

**Before generating code:**
- [ ] Define exactly what you need the code to do
- [ ] Specify the language, library versions, and data formats in your prompt
- [ ] Set temperature to 0.0–0.2 for deterministic output
- [ ] Set a fixed seed value if the API supports it

**After receiving code:**
- [ ] Read and understand every line
- [ ] Verify all imports and function calls against official documentation
- [ ] Test with known inputs and expected outputs
- [ ] Check for hard-coded values, magic numbers, and fabricated references
- [ ] Run static analysis (linter, type checker)

**Before committing to your project:**
- [ ] Add provenance comments indicating LLM use
- [ ] Replace generic comments with comments explaining *why*
- [ ] Add docstrings to all functions
- [ ] Refactor monolithic code into small, single-purpose functions
- [ ] Update your dependency file with pinned versions
- [ ] Save the full prompt and response to your LLM logs
- [ ] Commit to version control with a clear message

**Before publishing:**
- [ ] Include a LICENCE file
- [ ] Include a CITATION.cff file
- [ ] Write a README with installation, usage, and citation instructions
- [ ] Deposit in a DOI-issuing repository (Zenodo, DORA)
- [ ] Disclose LLM use in your publication and supplementary materials
- [ ] Ensure all LLM interaction logs are archived with the code
