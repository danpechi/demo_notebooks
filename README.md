# Prompt Registry, Optimization, and Migration

A comprehensive guide to optimizing and managing prompts when migrating between different language models using MLflow and Databricks Foundation Model APIs.

## Overview

This notebook demonstrates how to optimize prompts for different language models and manage them effectively using MLflow's prompt registry. Model switching is not as simple as swapping API endpoints—each model may require different prompt engineering to achieve optimal performance. This notebook shows you how to:

- Register and version prompts in MLflow
- Optimize prompts using training data with GepaPromptOptimizer
- Migrate prompts across different models (GPT-5, Gemma 3, GPT-OSS-20B)
- Use aliases to manage model-specific prompt versions

## Use Case

The notebook focuses on a text classification task: categorizing medical research abstracts into one of five categories:
- `BACKGROUND`
- `METHODS`
- `OBJECTIVE`
- `RESULTS`
- `CONCLUSIONS`

The goal is to optimize the model to output only the classification label without additional explanation.

## Prerequisites

### Required Access
- Databricks workspace
- Access to Databricks Foundation Model APIs
- Permissions to create catalogs and schemas in Unity Catalog

### Dependencies
```python
mlflow
databricks-sdk
dspy
openai
```

## Setup

### 1. Configure Your Environment

Update the catalog and schema variables in the notebook to match your environment:

```python
catalog = "main"  # Change to your catalog
schema = "default"  # Change to your schema
prompt_registry_name = "new_prompt_registry"
```

### 2. Install Dependencies

The notebook automatically installs required packages:
```python
%pip install --upgrade mlflow databricks-sdk dspy openai
```

## Workflow

### Step 1: Register Initial Prompt

Create a basic prompt template and register it in the MLflow prompt registry:

```python
prompt = mlflow.genai.register_prompt(
    name=prompt_location,
    template="classify this: {{query}}",
)
```

### Step 2: Test Baseline Performance

Evaluate how the model performs with the basic prompt. The initial prompt is intentionally minimal to demonstrate the improvement from optimization.

### Step 3: Optimize with Training Data

Use MLflow's `GepaPromptOptimizer` to automatically improve the prompt based on a dataset of examples with expected outputs:

- Provide input queries and expected classifications
- Define expectations and constraints (e.g., valid classification labels)
- Let the optimizer generate an improved prompt
- Register the optimized prompt as a new version

### Step 4: Migrate to Different Models

When switching models (e.g., from GPT-5 to Gemma 3), the optimized prompt may not perform well. The notebook shows how to:

1. Re-run optimization for the new model
2. Register the new optimized prompt as another version
3. Create aliases for model-specific prompts (e.g., `@gpt5`, `@gemma3`, `@gpt_oss_20b`)

### Step 5: Load Model-Specific Prompts

Use aliases to load the appropriate prompt version for each model:

```python
# For GPT-5
prompt = mlflow.genai.load_prompt(f"prompts:/{prompt_location}@gpt5")

# For Gemma 3
prompt = mlflow.genai.load_prompt(f"prompts:/{prompt_location}@gemma3")

# For GPT-OSS-20B
prompt = mlflow.genai.load_prompt(f"prompts:/{prompt_location}@gpt_oss_20b")
```

## Key Features

### MLflow Prompt Registry
- **Version Control**: Track all prompt iterations
- **Aliases**: Assign meaningful names to specific versions (e.g., `production`, `gpt5`, `gemma3`)
- **Centralized Management**: Single source of truth for all prompts

### GepaPromptOptimizer
- **Automated Optimization**: Improve prompts based on training data
- **Scoring**: Uses correctness metrics to evaluate prompt quality
- **Iterative Refinement**: Generates multiple candidate prompts and selects the best

### Model Flexibility
- Easy switching between models
- Model-specific prompt optimization
- Consistent interface across different models

## Training Data Format

The optimization requires training data in the following structure:

```python
dataset = [
    {
        "inputs": {"query": "Input text to classify"},
        "outputs": {"response": "EXPECTED_LABEL"},  # Optional
        "expectations": {
            "expected_response": "EXPECTED_LABEL",  # For exact match
            "expected_facts": ["Constraint description"]  # For validation rules
        }
    },
    # More examples...
]
```

## Expected Results

- **Before Optimization**: Basic classification with verbose explanations
- **After Optimization**: Concise, single-word classification labels
- **Cross-Model**: Consistent performance across different models with their optimized prompts

## Best Practices

1. **Start Simple**: Begin with a minimal prompt to establish a baseline
2. **Provide Diverse Examples**: Include representative samples from all classification categories
3. **Define Clear Expectations**: Specify both expected outputs and constraints
4. **Re-optimize Per Model**: Don't assume prompts transfer perfectly between models
5. **Use Aliases**: Organize prompts by model or use case for easy reference
6. **Iterate**: Run multiple optimization rounds if initial results aren't satisfactory

## Troubleshooting

### Common Issues

**Issue**: Optimization runs slowly or times out
- **Solution**: Reduce the size of your training dataset or number of optimization iterations

**Issue**: Model outputs unexpected formats
- **Solution**: Add more specific constraints in the `expected_facts` field

**Issue**: Cannot access prompt registry
- **Solution**: Verify your catalog and schema permissions in Unity Catalog

## Next Steps

- Experiment with different models and compare performance
- Expand the training dataset for better optimization results
- Deploy optimized prompts to production serving endpoints
- Set up automated evaluation pipelines to monitor prompt performance
- Explore advanced optimization parameters in GepaPromptOptimizer

## References

- [MLflow Prompt Engineering Documentation](https://mlflow.org/docs/latest/llms/prompt-engineering/index.html)
- [Databricks Foundation Model APIs](https://docs.databricks.com/en/machine-learning/foundation-models/index.html)
- [Unity Catalog](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)

## License

This notebook is provided as an example for educational and development purposes.
