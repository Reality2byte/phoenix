# Prompt Optimization



{% embed url="https://youtu.be/il5rQFjv3tM?si=W9gH89LzZNx_CoFu" %}

{% embed url="https://colab.research.google.com/github/Arize-ai/phoenix/blob/dbb46f76ef3292066858280d9c7718b674c6cebc/tutorials/prompts/prompt-optimization.ipynb" %}

This tutorial will use Phoenix to compare the performance of different prompt optimization techniques.

You'll start by creating an experiment in Phoenix that can house the results of each of your resulting prompts. Next you'll use a series of prompt optimization techniques to improve the performance of a jailbreak classification task. Each technique will be applied to the same base prompt, and the results will be compared using Phoenix.

The techniques you'll use are:

* **Few Shot Examples**: Adding a few examples to the prompt to help the model understand the task.
* **Meta Prompting**: Prompting a model to generate a better prompt based on previous inputs, outputs, and expected outputs.
* **Prompt Gradients**: Using the gradient of the prompt to optimize individual components of the prompt using embeddings.
* **DSPy Prompt Tuning**: Using DSPy, an automated prompt tuning library, to optimize the prompt.

⚠️ This tutorial requires and OpenAI API key.

Let's get started!

#### Setup Dependencies & Keys

```python
!pip install -q "arize-phoenix>=8.0.0" datasets
```

Next you need to connect to Phoenix. The code below will connect you to a Phoenix Cloud instance. You can also [connect to a self-hosted Phoenix instance](https://arize.com/docs/phoenix/deployment) if you'd prefer.

```python
import os
from getpass import getpass

os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "https://app.phoenix.arize.com"
if not os.environ.get("PHOENIX_CLIENT_HEADERS"):
    os.environ["PHOENIX_CLIENT_HEADERS"] = "api_key=" + getpass("Enter your Phoenix API key: ")

if not os.environ.get("OPENAI_API_KEY"):
    os.environ["OPENAI_API_KEY"] = getpass("Enter your OpenAI API key: ")
```

#### Load Dataset into Phoenix

Since we'll be running a series of experiments, we'll need a dataset of test cases that we can run each time. This dataset will be used to test the performance of each prompt optimization technique.

```python
from datasets import load_dataset

ds = load_dataset("jackhhao/jailbreak-classification")["train"]
ds = ds.to_pandas().sample(50)
ds.head()
```

```python
import uuid

import phoenix as px
from phoenix.client import Client as PhoenixClient

unique_id = uuid.uuid4()

# Upload the dataset to Phoenix
dataset = px.Client().upload_dataset(
    dataframe=ds,
    input_keys=["prompt"],
    output_keys=["type"],
    dataset_name=f"jailbreak-classification-{unique_id}",
)
```

Next, you can define a base template for the prompt. We'll also save this template to Phoenix, so it can be tracked, versioned, and reused across experiments.

```python
from openai import OpenAI
from openai.types.chat.completion_create_params import CompletionCreateParamsBase

from phoenix.client.types import PromptVersion

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {
            "role": "system",
            "content": "You are an evaluator that decides whether a given prompt is a jailbreak risk. Only output 'benign' or 'jailbreak', no other words.",
        },
        {"role": "user", "content": "{{prompt}}"},
    ],
)

prompt_identifier = "jailbreak-classification"

prompt = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="A prompt for classifying whether a given prompt is a jailbreak risk.",
    version=PromptVersion.from_openai(params),
)
```

You should now see that prompt in Phoenix:

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/prompts-screenshot-prompt-optimization-2.png)

Next you'll need a task and evaluator for the experiment. A task is a function that will be run across each example in the dataset. The task is also the piece of your code that you'll change between each run of the experiment. To start off, the task is simply a call to GPT 3.5 Turbo with a basic prompt.

You'll also need an evaluator that will be used to test the performance of the task. The evaluator will be run across each example in the dataset after the task has been run. Here, because you have ground truth labels, you can use a simple function to check if the output of the task matches the expected output.

```python
def test_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(**prompt.format(variables={"prompt": input["prompt"]}))
    return resp.choices[0].message.content.strip()


def evaluate_response(output, expected):
    return output.lower() == expected["type"].lower()
```

You can also instrument your code to send all models calls to Phoenix. This isn't necessary for the experiment to run, but it does mean all your experiment task runs will be tracked in Phoenix. The overall experiment score and evaluator runs will be tracked regardless of whether you instrument your code or not.

```python
from openinference.instrumentation.openai import OpenAIInstrumentor

from phoenix.otel import register

tracer_provider = register(project_name="prompt-optimization")
OpenAIInstrumentor().instrument(tracer_provider=tracer_provider)
```

Now you can run the initial experiment. This will be the base prompt that you'll be optimizing.

```python
import nest_asyncio

from phoenix.experiments import run_experiment

nest_asyncio.apply()

initial_experiment = run_experiment(
    dataset,
    task=test_prompt,
    evaluators=[evaluate_response],
    experiment_description="Initial base prompt",
    experiment_name="initial-prompt",
    experiment_metadata={"prompt": "prompt_id=" + prompt.id},
)
```

You should now see the initial experiment results in Phoenix:

![1st experiment results](https://storage.googleapis.com/arize-phoenix-assets/assets/images/prompts-nb-experiment.png)

## Prompt Optimization Technique #1: Few Shot Examples

One common prompt optimization technique is to use few shot examples to guide the model's behavior.

Here you can add few shot examples to the prompt to help improve performance. Conviently, the dataset you uploaded in the last step contains a test set that you can use for this purpose.

```python
from datasets import load_dataset

ds_test = load_dataset("jackhhao/jailbreak-classification")[
    "test"
]  # this time, load in the test set instead of the training set
few_shot_examples = ds_test.to_pandas().sample(10)
```

Define a new prompt that includes the few shot examples. Prompts in Phoenix are automatically versioned, so saving the prompt with the same name will create a new version that can be used.

```python
few_shot_template = """
You are an evaluator that decides whether a given prompt is a jailbreak risk. Only output "benign" or "jailbreak", no other words.

Here are some examples of prompts and responses:

{examples}
"""

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {"role": "system", "content": few_shot_template.format(examples=few_shot_examples)},
        {"role": "user", "content": "{{prompt}}"},
    ],
)

few_shot_prompt = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="Few shot prompt",
    version=PromptVersion.from_openai(params),
)
```

You'll notice you now have a new version of the prompt in Phoenix:

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/prompt-versioning-nb.png)

Define a new task with your new prompt:

```python
def test_prompt(input):
    client = OpenAI()
    prompt_vars = {"prompt": input["prompt"]}
    resp = client.chat.completions.create(**few_shot_prompt.format(variables=prompt_vars))
    return resp.choices[0].message.content.strip()
```

Now you can run another experiment with the new prompt. The dataset of test cases and the evaluator will be the same as the previous experiment.

```python
few_shot_experiment = run_experiment(
    dataset,
    task=test_prompt,
    evaluators=[evaluate_response],
    experiment_description="Prompt Optimization Technique #1: Few Shot Examples",
    experiment_name="few-shot-examples",
    experiment_metadata={"prompt": "prompt_id=" + few_shot_prompt.id},
)
```

## Prompt Optimization Technique #2: Meta Prompting

Meta prompting involves prompting a model to generate a better prompt, based on previous inputs, outputs, and expected outputs.

The experiment from round 1 serves as a great starting point for this technique, since it has each of those components.

```python
# Access the experiment results from the first round as a dataframe
ground_truth_df = initial_experiment.as_dataframe()

# Sample 10 examples to use as meta prompting examples
ground_truth_df = ground_truth_df[:10]

# Create a new column with the examples in a single string
ground_truth_df["example"] = ground_truth_df.apply(
    lambda row: f"Input: {row['input']}\nOutput: {row['output']}\nExpected Output: {row['expected']}",
    axis=1,
)
ground_truth_df.head()
```

Now construct a new prompt that will be used to generate a new prompt.

```python
meta_prompt = """
You are an expert prompt engineer. You are given a prompt, and a list of examples.

Your job is to generate a new prompt that will improve the performance of the model.

Here are the examples:

{examples}

Here is the original prompt:

{prompt}

Here is the new prompt:
"""

original_base_prompt = (
    prompt.format(variables={"prompt": "example prompt"}).get("messages")[0].get("content")
)

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[
        {
            "role": "user",
            "content": meta_prompt.format(
                prompt=original_base_prompt, examples=ground_truth_df["example"].to_string()
            ),
        }
    ],
)
new_prompt = response.choices[0].message.content.strip()
```

```python
new_prompt
```

Now save that as a prompt in Phoenix:

```python
if r"\{examples\}" in new_prompt:
    new_prompt = new_prompt.format(examples=few_shot_examples)

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {"role": "system", "content": new_prompt},
        {"role": "user", "content": "{{prompt}}"},
    ],
)

meta_prompt_result = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="Meta prompt result",
    version=PromptVersion.from_openai(params),
)
```

#### Run this new prompt through the same experiment

Redefine the task, using the new prompt.

```python
def test_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **meta_prompt_result.format(variables={"prompt": input["prompt"]})
    )
    return resp.choices[0].message.content.strip()
```

```python
meta_prompting_experiment = run_experiment(
    dataset,
    task=test_prompt,
    evaluators=[evaluate_response],
    experiment_description="Prompt Optimization Technique #2: Meta Prompting",
    experiment_name="meta-prompting",
    experiment_metadata={"prompt": "prompt_id=" + meta_prompt_result.id},
)
```

## Prompt Optimization Technique #3: Prompt Gradient Optimization

Prompt gradient optimization is a technique that uses the gradient of the prompt to optimize individual components of the prompt using embeddings. It involves:

1. Converting the prompt into an embedding.
2. Comparing the outputs of successful and failed prompts to find the gradient direction.
3. Moving in the gradient direction to optimize the prompt.

Here you'll define a function to get embeddings for prompts, and then use that function to calculate the gradient direction between successful and failed prompts.

```python
import numpy as np


# First we'll define a function to get embeddings for prompts
def get_embedding(text):
    client = OpenAI()
    response = client.embeddings.create(model="text-embedding-ada-002", input=text)
    return response.data[0].embedding


# Function to calculate gradient direction between successful and failed prompts
def calculate_prompt_gradient(successful_prompts, failed_prompts):
    # Get embeddings for successful and failed prompts
    successful_embeddings = [get_embedding(p) for p in successful_prompts]
    failed_embeddings = [get_embedding(p) for p in failed_prompts]

    # Calculate average embeddings
    avg_successful = np.mean(successful_embeddings, axis=0)
    avg_failed = np.mean(failed_embeddings, axis=0)

    # Calculate gradient direction
    gradient = avg_successful - avg_failed
    return gradient / np.linalg.norm(gradient)


# Get successful and failed examples from our dataset
successful_examples = (
    ground_truth_df[ground_truth_df["output"] == ground_truth_df["expected"].get("type")]["input"]
    .apply(lambda x: x["prompt"])
    .tolist()
)
failed_examples = (
    ground_truth_df[ground_truth_df["output"] != ground_truth_df["expected"].get("type")]["input"]
    .apply(lambda x: x["prompt"])
    .tolist()
)

# Calculate the gradient direction
gradient = calculate_prompt_gradient(successful_examples[:5], failed_examples[:5])


# Function to optimize a prompt using the gradient
def optimize_prompt(base_prompt, gradient, step_size=0.1):
    # Get base embedding
    base_embedding = get_embedding(base_prompt)

    # Move in gradient direction
    optimized_embedding = base_embedding + step_size * gradient

    # Use GPT to convert the optimized embedding back to text
    client = OpenAI()
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {
                "role": "system",
                "content": "You are helping to optimize prompts. Given the original prompt and its embedding, generate a new version that maintains the core meaning but moves in the direction of the optimized embedding.",
            },
            {
                "role": "user",
                "content": f"Original prompt: {base_prompt}\nOptimized embedding direction: {optimized_embedding[:10]}...\nPlease generate an improved version that moves in this embedding direction.",
            },
        ],
    )
    return response.choices[0].message.content.strip()


# Test the gradient-based optimization
gradient_prompt = optimize_prompt(original_base_prompt, gradient)
```

```python
gradient_prompt
```

```python
if r"\{examples\}" in gradient_prompt:
    gradient_prompt = gradient_prompt.format(examples=few_shot_examples)

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {
            "role": "system",
            "content": gradient_prompt,
        },  # if your meta prompt includes few shot examples, make sure to include them here
        {"role": "user", "content": "{{prompt}}"},
    ],
)

gradient_prompt = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="Gradient prompt result",
    version=PromptVersion.from_openai(params),
)
```

#### Run experiment with gradient-optimized prompt

Redefine the task, using the new prompt.

```python
def test_gradient_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **gradient_prompt.format(variables={"prompt": input["prompt"]})
    )
    return resp.choices[0].message.content.strip()
```

```python
gradient_experiment = run_experiment(
    dataset,
    task=test_gradient_prompt,
    evaluators=[evaluate_response],
    experiment_description="Prompt Optimization Technique #3: Prompt Gradients",
    experiment_name="gradient-optimization",
    experiment_metadata={"prompt": "prompt_id=" + gradient_prompt.id},
)
```

## Prompt Optimization Technique #4: Prompt Tuning with DSPy

Finally, you can use an optimization library to optimize the prompt, like DSPy. [DSPy](https://github.com/stanfordnlp/dspy) supports each of the techniques you've used so far, and more.

```python
!pip install -q dspy openinference-instrumentation-dspy
```

DSPy makes a series of calls to optimize the prompt. It can be useful to see these calls in action. To do this, you can instrument the DSPy library using the OpenInference SDK, which will send all calls to Phoenix. This is optional, but it can be useful to have.

```python
from openinference.instrumentation.dspy import DSPyInstrumentor

DSPyInstrumentor().instrument(tracer_provider=tracer_provider)
```

Now you'll setup the DSPy language model and define a prompt classification task.

```python
# Import DSPy and set up the language model
import dspy

# Configure DSPy to use OpenAI
turbo = dspy.LM(model="gpt-3.5-turbo")
dspy.settings.configure(lm=turbo)


# Define the prompt classification task
class PromptClassifier(dspy.Signature):
    """Classify if a prompt is benign or jailbreak."""

    prompt = dspy.InputField()
    label = dspy.OutputField(desc="either 'benign' or 'jailbreak'")


# Create the basic classifier
classifier = dspy.Predict(PromptClassifier)
```

Your classifier can now be used to make predictions as you would a normal LLM. It will expect a `prompt` input and will output a `label` prediction.

```python
classifier(prompt=ds.iloc[0].prompt)
```

However, DSPy really shines when it comes to optimizing prompts. By defining a metric to measure successful runs, along with a training set of examples, you can use one of many different optimizers built into the library.

In this case, you'll use the `MIPROv2` optimizer to find the best prompt for your task.

```python
def validate_classification(example, prediction, trace=None):
    return example["label"] == prediction["label"]


# Prepare training data from previous examples
train_data = []
for _, row in ground_truth_df.iterrows():
    example = dspy.Example(
        prompt=row["input"]["prompt"], label=row["expected"]["type"]
    ).with_inputs("prompt")
    train_data.append(example)

tp = dspy.MIPROv2(metric=validate_classification, auto="light")
optimized_classifier = tp.compile(classifier, trainset=train_data)
```

DSPy takes care of our prompts in this case, however you could still save the resulting prompt value in Phoenix:

```python
params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {
            "role": "system",
            "content": optimized_classifier.signature.instructions,
        },  # if your meta prompt includes few shot examples, make sure to include them here
        {"role": "user", "content": "{{prompt}}"},
    ],
)

dspy_prompt = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="DSPy prompt result",
    version=PromptVersion.from_openai(params),
)
```

#### Run experiment with DSPy-optimized classifier

Redefine the task, using the new prompt.

```python
# Create evaluation function using optimized classifier
def test_dspy_prompt(input):
    result = optimized_classifier(prompt=input["prompt"])
    return result.label
```

```python
# Run experiment with DSPy-optimized classifier
dspy_experiment = run_experiment(
    dataset,
    task=test_dspy_prompt,
    evaluators=[evaluate_response],
    experiment_description="Prompt Optimization Technique #4: DSPy Prompt Tuning",
    experiment_name="dspy-optimization",
    experiment_metadata={"prompt": "prompt_id=" + dspy_prompt.id},
)
```

## Prompt Optimization Technique #5: DSPy with GPT-4o

In the last example, you used GPT-3.5 Turbo to both run your pipeline, and optimize the prompt. However, you can also use a different model to optimize the prompt, and a different model to run your pipeline.

It can be useful to use a more powerful model for your optimization step, and a cheaper or faster model for your pipeline.

Here you'll use GPT-4o to optimize the prompt, and keep GPT-3.5 Turbo as your pipeline model.

```python
prompt_gen_lm = dspy.LM("gpt-4o")
tp = dspy.MIPROv2(
    metric=validate_classification, auto="light", prompt_model=prompt_gen_lm, task_model=turbo
)
optimized_classifier_using_gpt_4o = tp.compile(classifier, trainset=train_data)
```

#### Run experiment with DSPy-optimized classifier using GPT-4o

Redefine the task, using the new prompt.

```python
# Create evaluation function using optimized classifier
def test_dspy_prompt(input):
    result = optimized_classifier_using_gpt_4o(prompt=input["prompt"])
    return result.label
```

```python
# Run experiment with DSPy-optimized classifier
dspy_experiment_using_gpt_4o = run_experiment(
    dataset,
    task=test_dspy_prompt,
    evaluators=[evaluate_response],
    experiment_description="Prompt Optimization Technique #5: DSPy Prompt Tuning with GPT-4o",
    experiment_name="dspy-optimization-gpt-4o",
    experiment_metadata={"prompt": "prompt_id=" + dspy_prompt.id},
)
```

## Results

And just like that, you've run a series of prompt optimization techniques to improve the performance of a jailbreak classification task, and compared the results using Phoenix.

You should have a set of experiments that looks like this:

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/prompt-optimization-experiment-screenshot.png)

From here, you can check out more [examples on Phoenix](https://arize.com/docs/phoenix/notebooks), and if you haven't already, [please give us a star on GitHub!](https://github.com/Arize-ai/phoenix) ⭐️
