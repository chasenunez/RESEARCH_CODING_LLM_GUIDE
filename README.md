*A note for readers:* This is mostly a collection of ideas. It should not be viewed as authoratative, nor accepted without critical thinking. This repo is open for additions, subtractions, ammendments, and disagreement. Please feel free to make a productive PR with your thoughts.

[![DOI](https://zenodo.org/badge/1392019667.svg)](https://doi.org/10.5281/zenodo.23008504)


# Large Language Models for Research Code: A Practical Guide

Large language models are, by design, not compatible with reproducible coding. An LLM is non-deterministic: the same prompt can give different outputs. You cannot see its internal states, and developers often do not disclose the training data. Its outputs can change with time, between versions, and between two prompts that are the same. Parameter adjustments do not change this.

An LLM generates text that has a high statistical probability. It does not make sure that the text is correct. This difference is very important for research. An LLM has these limits:

- It can write code that looks correct and runs without errors, but gives incorrect results.
- It frequently writes function signatures, package names, and application programming interface (API) parameters that no library contains. The name for such output is "hallucination."
- It cannot see your runtime environment. It does not know which library versions you installed.
- It does not know your scientific purpose. It selects tokens that have a high probability. It does not make sure that your analysis is correct.
- It does not tell you when it is not sure. A correct output and a hallucination look the same.
- If you tell it to examine the code that it wrote, it almost always tells you that the code is correct.
- Commercial LLM APIs do not guarantee deterministic output: even with [a temperature of 0](https://www.ibm.com/think/topics/llm-temperature), small variations can occur between runs due to hardware, load-balancing, and silent model updates.

But the LLM is not responsible for reproducible code. **You are.**

An LLM is a tool. It is in the same category as a calculator, a compiler, or a note that a colleague writes on a whiteboard. Researchers have always used tools that they did not fully control, but we must do the work to carefully and critically use these tools. 

An LLM does not make your research more reproducible or less reproducible. Your control of the input and the output does. For the input, record the prompts, the instructions, the model parameters, and the tools that you used. For the output, examine it, do tests on it, record it, and make it transparent.

An LLM is not always the correct tool. If you know how to write the function, it can be faster and safer to write it without an LLM. For some numerical algorithms, you must show that each line is scientifically correct. For such algorithms, it can be more work to examine LLM-generated code than to write the code.

This guide shows you how to use LLMs to help with your code. It also shows you how to keep your research software reproducible, transparent, and easy to use again. In some conditions, an LLM can make these qualities better. The guide has three phases: before you write a prompt, during the work with generated code, and before publication.

The basic method has three parts: **write small prompts, examine all output, and record each step.**

---

## Phase 1: Before You Write a Prompt

### Divide the Task

This is the most important rule for reproducible LLM code: **do not tell an LLM to write a full program in one prompt.** An example of such a prompt is "create a program that analyses my hydrology data and produces a report." A large, open prompt increases the risk of hallucinations. The model makes assumptions that it does not tell you about, and it adds dependencies that are not available. The result is monolithic code, which is not easy to examine, to change, or to use again.

As an alternative, divide your task into functions. Write one prompt with accurate specifications for each function. This is good software engineering practice. It also makes LLM output more accurate.

### One Function, One Prompt

For each function, your prompt must give four items:

1. **The input:** the types, shapes, and units of the data that goes in.
2. **The operation:** the algorithm or operation, with a reference if possible.
3. **The output:** the type, shape, and description of the data that comes out.
4. **The possible errors:** the edge cases and the correct response to each.

Do not write "write me a function to do X." Give the library versions that you use, the data structures, and an example of the input and output. When your prompt is more accurate, the risk of hallucinations is lower. Tell the model to give an explanation of its method before it writes code. Then you can see logic errors and correct them before the model generates code.

### A Bad Prompt and a Good Prompt

**Bad: vague, open-ended, invites hallucination:**

> "Write a Python program that reads my CSV files, cleans the data, does some statistical analysis, and makes nice plots."

This prompt gives you a long script that is full of assumptions. The assumptions are about your file structure, column names, data filters, statistical methods, and plot format. You must correct many of them. It is slow work to isolate them in a monolithic script.

**Good: accurate, with clear limits, and for one function only**

> "I need a Python function called `load_sensor_data` that takes a single argument `filepath` (a string pointing to a CSV file). The CSV has columns 'timestamp' (ISO 8601 format), 'temperature_c' (float, degrees Celsius), and 'conductivity_us' (float, microsiemens/cm). The function should parse the timestamp column into pandas datetime, drop any rows where temperature or conductivity is NaN, and return a pandas DataFrame with those three columns. Use only pandas (version 2.2.x). Raise a FileNotFoundError if the file does not exist and a ValueError if expected columns are missing."

This prompt gives the model tight limits. The output will be short and easy to examine. You can do a test of it with a known input file.

### Write a System Prompt for the Project

When you divide a task into small steps, you write many prompts. Some general instructions are applicable to all of them. Examples are the programming language, the code structure, and the rules for comments. Do not write these instructions again in each prompt. Collect them in one system prompt for the full project.

A system prompt is a set of general instructions that the model receives before each task prompt. Different tools have different names for it. Examples are "system prompt," "custom instructions," and "project instructions." If your tool does not have this function, put the text at the start of each prompt.

A project system prompt has these effects:

- The outputs of all prompts have the same structure.
- Each task prompt is shorter. It contains only the instructions for that task, for example the operation that the function must do.
- The general instructions are in one file. You record them one time, and all your prompts use the same version.

This is an example of a system prompt for a Python project that contains many functions:

```text
Generate a function in Python 3.XX. Structure the output code as follows:
1. Write a header that states what the code does, the date of generation,
   and the specific model that generated the code.
2. List all dependencies (Python packages) that the function uses.
3. Specify all global variables that the code needs to run. Do not put
   "magic" values directly in the code. Instead, define them as global
   variables and refer to them by variable name in the rest of the code.
4. Give the code for the function.
5. Include a docstring under the function name. The docstring must give the
   description, the inputs (names and object types), the outputs (types),
   and all errors that the function can raise.
6. Include comments that a person can read easily. The comments must
   explain what the code does and why.
```

Replace `Python 3.XX` with a relevant coding environment for your project. Keep the system prompt in a file in version control. Record it in your logs together with each task prompt. If you change the system prompt, record the date and the cause of the change.

Compare each generated header with your log. The model can write an incorrect date or an incorrect model name in the header. Your log is the correct source.

### Set Model Parameters for Reproducibility

An API lets your code send prompts directly to a model, without a chat window. When you use an LLM through an API, [set the temperature parameter](https://www.ibm.com/think/topics/llm-temperature) as low as possible (0.0 to 0.2) for code generation. At a low temperature, the model selects the tokens that have the highest probability. This decreases the quantity of unusual output, which is frequently incorrect.

Do not apply frequency penalties or presence penalties for code generation. Correct code contains the same variable names, keywords, and structural patterns many times. If the API has a seed parameter, set a seed and record it.

If your research includes LLM-generated or LLM-assisted code, you must record the conditions of its generation. Then other researchers can examine your work and, where possible, reproduce it. [Baltes et al. (2025)](https://arxiv.org/abs/2508.15503) wrote guidelines for empirical software engineering studies that use LLMs. Their guidelines tell researchers to include model versions, configurations, prompts, and interaction logs in their reports. The items that follow agree with these guidelines.

**Model identity and version.** Record the full model name and version string (for example, `gpt-4-0125-preview` or `claude-sonnet-4-20250514`). For a model that you run on a local computer, record the Hugging Face model card identifier. Also record the quantization format, the inference framework (for example, Ollama, vLLM, or llama.cpp), and the framework version.

**All sampling parameters.** Record at minimum: temperature, top_p, top_k (if applicable), max_tokens, all penalty parameters, and the seed value if you set one. If you used the default settings, write that in your log. Default settings change between versions.

**The interface and its tools.** Record how you got access to the model: a chat window, a code editor assistant, a coding agent, or an API. Record the name and version of that interface. Such interfaces can add instructions and parameters to your prompt.

**The system fingerprint or its equivalent.** Some providers (for example, OpenAI) return a `system_fingerprint` with each API response. Record it. It helps you identify changes in the infrastructure of the provider.

**The date of each query.** Providers can update commercial models and not tell you. Thus, the date of your query is part of the provenance.

**The full prompt and the full response.** Save the full prompt text, together with the system prompt. If you sent more than one prompt in the same conversation, save the full conversation log. Save the full model output, not only the code that you used. Prompts are part of your experimental method. The full response shows other researchers the code that the model generated and the changes that you made.

### Record Your Queries Automatically

A software development kit (SDK) is a library that a provider supplies. With an SDK, your code can call the API of the provider in a small number of lines. Write a wrapper function around your SDK calls. The wrapper must record the items above automatically and save them to a structured file (for example, JSON or YAML). This is an example:

```python
# Minimal record of one LLM API call, kept for reproducibility.
# SDK = software development kit: the provider's library for calling its API.
# Each provider's SDK returns a slightly different response object,
# so adapt the fields below to yours.

import datetime
import json


def log_llm_call(prompt, response, model, params, output_path,
                 system_prompt=None, system_fingerprint=None):
    """
    Save a complete record of one LLM interaction for reproducibility.

    Args:
        prompt: The full task prompt text sent to the model.
        response: The complete response text from the model.
        model: Model identifier string (e.g., 'gpt-4-0125-preview').
        params: Dict of sampling parameters (temperature, top_p, seed, etc.).
        output_path: Path to the log file (JSON Lines: one record per line).
        system_prompt: The project system prompt, if you used one.
        system_fingerprint: The provider's infrastructure identifier,
            if the API returned one.
    """
    record = {
        # Timezone-aware UTC, so logs from different machines sort correctly.
        "timestamp": datetime.datetime.now(datetime.timezone.utc).isoformat(),
        "model": model,
        "parameters": params,
        "system_prompt": system_prompt,
        "prompt": prompt,
        "response_text": response,
        "system_fingerprint": system_fingerprint,
    }
    # Append instead of overwrite: earlier records are part of the research
    # record, and one record per line keeps the file easy to read back.
    with open(output_path, "a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

Store these logs with your code in version control. They are part of your research record.

---

## Phase 2: During the Work

### An Example: An Analysis in Four Steps

In this example, you must process sensor data from a field station. The steps are: load the data, remove the outliers, calculate the daily means, and make a plot of the result. Do not write one prompt. Write four.

**Prompt 1: Load the data**
> "Write a Python function `load_sensor_data(filepath: str) -> pd.DataFrame` that reads a CSV with columns 'timestamp', 'temperature_c', 'conductivity_us'. Parse timestamps as datetime, drop rows with NaN in either measurement column. Return the cleaned DataFrame. Use pandas 2.2.x only. Raise FileNotFoundError if path is invalid, ValueError if columns are missing."

**Prompt 2: Remove the outliers**
> "Write a Python function `remove_outliers(df: pd.DataFrame, column: str, n_sigma: float = 3.0) -> pd.DataFrame` that removes rows where the value in `column` is more than `n_sigma` standard deviations from the mean. Return a new DataFrame (do not modify the input). Raise KeyError if the column does not exist. Use only pandas and numpy."

**Prompt 3: Calculate the daily means**
> "Write a Python function `daily_means(df: pd.DataFrame, timestamp_col: str = 'timestamp', value_cols: list[str] = None) -> pd.DataFrame` that groups by calendar date (derived from the timestamp column) and returns the mean of each value column. If value_cols is None, use all numeric columns. The returned DataFrame should have 'date' as the index. Use pandas only."

**Prompt 4: Make a plot of the result**
> "Write a Python function `plot_timeseries(df: pd.DataFrame, y_columns: list[str], title: str, output_path: str) -> None` that creates a line plot with date on the x-axis and one line per column in y_columns. Use matplotlib 3.9.x. Label axes with column names. Save the figure to output_path at 300 DPI. Do not call plt.show()."

You can do tests on each of these functions independently. You can also examine and replace each function independently. If the outlier function is incorrect, you correct one function, not a full script.

### Verify Each Function Before Moving On

After the LLM returns a function, do these steps before you write the next prompt:

1. **Read the code.** Make sure that you know the function of each line. If a part is not clear, tell the LLM to give an explanation. Then compare the explanation with the library documentation. Do not tell the same LLM to examine the explanation that it gave.
2. **Examine the imports.** Make sure that the function uses only the libraries that you specified. Make sure that the function names and signatures are in the documentation for the library version that you use.
3. **Write a short test.** Make a small test input for which you know the correct output. Run the function with this input. For numerical or analytical code, use an example that has a known correct result.
4. **Run static analysis.** Use a linter and a type checker (for example, mypy, pylint, or ESLint) to find problems in the code.

### Code Structure: What LLMs Get Wrong

The rules of good software engineering are the same for code from a person and code from an LLM. But LLMs frequently do not obey some of these rules. Look for the problems that follow each time.

**Monolithic functions.** LLMs frequently write one large function that reads data, processes it, and makes a plot of the results. Divide it into smaller functions that each do one task.

**Magic numbers.** LLMs frequently put values directly in the code. Examples are thresholds, file paths, and column indices. Replace these values with named constants or configuration parameters.

**Missing dependency specifications.** Record each external library in a dependency file (for example, requirements.txt, environment.yml, or renv.lock) with a pinned version number. LLMs almost always write dependency specifications that are not full or not correct. You must do this work.

**No error handling.** LLMs frequently write code that operates correctly only when all inputs are correct. Add input validation and clear error messages.

**Code that is not clear.** LLMs can write code that is short but not clear. Examples are list comprehensions that contain too many operations, unusual built-in functions, and optimizations that are not necessary. Clear code is better than short code. A colleague who reads your code in three years must know its function without the LLM.

### Assemble the Functions into a Pipeline

When your functions have tests and documentation, write a short pipeline script that calls them in sequence. This script is the part that *you* write. It contains your scientific workflow and your decisions. The LLM helped with the parts. You are the architect.

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

# --- Configuration (all tunable parameters in one place) ---
INPUT_FILE = "data/station_alpha_2025.csv"
OUTLIER_THRESHOLD = 3.0  # standard deviations (see the Methods section of the paper)
VALUE_COLUMNS = ["temperature_c", "conductivity_us"]
OUTPUT_PLOT = "figures/daily_means.png"

# --- Execution ---
df = load_sensor_data(INPUT_FILE)
df = remove_outliers(df, column="temperature_c", n_sigma=OUTLIER_THRESHOLD)
df = remove_outliers(df, column="conductivity_us", n_sigma=OUTLIER_THRESHOLD)
daily = daily_means(df, value_cols=VALUE_COLUMNS)
plot_timeseries(daily, y_columns=VALUE_COLUMNS,
                title="Daily Mean Sensor Readings, Station Alpha",
                output_path=OUTPUT_PLOT)
```

This pipeline is easy to read and to examine. It agrees with the FAIR principles (findable, accessible, interoperable, reusable). You can see all the parameters. Each step is a named function that you can examine independently. The configuration and the logic are in different parts of the script.

### Commenting and Documenting LLM-Assisted Code

Comments in LLM-assisted code have more functions than comments in code that a person wrote. They must tell the reader *what* the code does. They must also tell the reader *where the code came from* and *how carefully you examined it*.

**Provenance comments.** If an LLM wrote or changed a large part of a file or function, add a provenance comment:

```python
# ----- Provenance -----
# This function was initially generated by Claude (claude-sonnet-4-20250514)
# on 2026-04-15 in response to the prompt stored in ./llm_logs/sort_func.json.
# It was subsequently reviewed, tested, and modified by [Your Name].
# Modifications: added input validation, fixed off-by-one error in line 34,
# replaced fabricated 'np.fastsort' call with np.sort.
# -----------------------
```

This comment has three functions:

- It obeys the transparency requirements of publishers and funders. For example, the ACM Policy on Authorship tells authors to fully disclose generative AI tools that they used to make content.
- It tells readers where the code came from. You are one of these readers when you open the file again in a year.
- It shows which parts you must possibly examine again.

**Inline comments: tell the reader *why*, not *what*.** LLMs frequently write comments that give the code again in words, for example "increment counter by 1." Such comments do not give the purpose of the line. Replace them with comments that give the purpose, the assumptions, and the related science:

```python
# WRONG: restates the code.
x = x + 1  # increment x by one

# RIGHT: explains the reasoning.
x = x + 1  # shift to 1-based indexing to match Table 2 in [your reference]
```

**Docstrings.** Each function must have a docstring. The docstring gives the purpose, the parameters, the return value, and the exceptions of the function. If the function is not easy to use, the docstring also gives an example.

Use the same format for all docstrings (for example, NumPy style or Google style for Python). An LLM can write a draft of a docstring. But you must make sure that the parameter descriptions agree with the function behavior and that the assumptions are correct.

```python
def calculate_flux(concentration, velocity, cross_section):
    """
    Calculate mass flux through a cross-section.

    Implements Eq. 3 from [your reference].
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

### Summary of the Function-by-Function Method

| Do not do this | Do this |
|---|---|
| "Write a program that does X" | "Write a function that takes Y, does Z, returns Q" |
| Write one long prompt for the full task | Write one prompt for each function, with input and output specifications |
| Accept a 200-line script without changes | Do a test of each function independently before you assemble the functions |
| Let the LLM select the libraries | Give the library and its version in the prompt |
| Send the prompt "is this correct?" | Do tests with known inputs and their correct outputs |
| Accept LLM-generated comments without changes | Write the comments again to give *why*, not *what* |

---

## Phase 3: Before Publication

### Version Control

Use Git or a different version control system during the full project, not only at the end. Version control is possibly more important for reproducibility than an LLM parameter. It gives you a full version history of your code, and it lets you easily remove an incorrect change. It is also the correct location for the LLM logs and the provenance comments. Make commits frequently, from the start of the project. Write clear commit messages that show which work is your work and which work is LLM-assisted.

### Transparency and Disclosure

Many publishers and funders have a disclosure requirement for generative AI. For example, the ACM Policy on Authorship tells authors to fully disclose generative AI tools that they used to make content. Where there is no such requirement, transparency is good scientific practice.

In your publication or supplementary materials, give this information:

- The tasks for which you used an LLM (for example, code generation, debugging, or documentation)
- The model and its version
- A statement that you examined the output, did tests on it, and are responsible for it.

In your code repository, do these steps:

1. Put the LLM logs (prompts and responses) in a dedicated directory.
2. Add provenance comments to code that an LLM generated or changed.
3. Write a description of your verification procedure in the README file or in a CONTRIBUTING file.

### Responsibility

When you use an LLM, the responsibility for your code stays with you. You are the author. If the LLM caused a bug, an incorrect assumption, or a dependency that is a hallucination, you are responsible. This is a rule of research ethics, and it is also a fact of research practice. Reviewers, collaborators, and users come to you, not to the LLM, when the software has a problem.

---

## Checklist

Use this checklist when you add LLM-generated code to a research project.

**Before you generate code:**
- [ ] Write an accurate description of the function of the code.
- [ ] Write or update the project system prompt.
- [ ] Give the language, library versions, and data formats in your prompt.
- [ ] Set the temperature to a value from 0.0 to 0.2 to decrease the variation in the output.
- [ ] Set a seed value if the API has a seed parameter.

**After you receive code:**
- [ ] Read each line and make sure that you know its function.
- [ ] Compare all imports and function calls with the library documentation.
- [ ] Do tests with known inputs and their correct outputs.
- [ ] Look for hard-coded values, magic numbers, and references that are hallucinations.
- [ ] Run static analysis (linter, type checker).

**Before you add the code to your project:**
- [ ] Add provenance comments that show where you used an LLM.
- [ ] Replace comments that give *what* with comments that give *why*.
- [ ] Add docstrings to all functions.
- [ ] Divide monolithic code into small functions that each do one task.
- [ ] Update your dependency file with pinned versions.
- [ ] Save the full prompt, the system prompt, and the response to your LLM logs.
- [ ] Make a commit in version control with a clear message.

**Before publication:**
- [ ] Include a LICENSE file.
- [ ] Include a CITATION.cff file.
- [ ] Write a README file with instructions for installation, operation, and citation.
- [ ] Put the code in a repository that gives a digital object identifier (DOI), for example Zenodo.
- [ ] Tell your readers in the publication and supplementary materials that you used an LLM.
- [ ] Make sure that all LLM logs are in the archive together with the code.
