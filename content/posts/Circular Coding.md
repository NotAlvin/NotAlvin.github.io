+++
title = 'Replicating Paper Performance Part 1'
date = 2024-05-02T00:42:24+08:00
draft = false
math = false
+++

## Introduction
Replicating the performance of scientific papers is vital for ensuring the integrity and reliability of research findings. This is especially relevant in the machine learning field, where with new Generative AI papers coming out every day validating the results of previous studies allow us to verify the robustness of their conclusions. By independently reproducing experimental procedures and analyses, data scientists can uncover potential errors, biases, or limitations in the original work, thereby strengthening the validity of the scientific knowledge base.

Moreover, replication fosters transparency and accountability within the scientific community, promoting trust and confidence in the reliability of research outcomes. Going back to the topic of our previous article on [Quantifying Agent Performance Optimization](https://notalvin.github.io/posts/agent-ablation/) today we'll be trying to replicate the outcomes of [AgentCoder](https://arxiv.org/abs/2312.13010), a recently published paper on Multi-Agent-based Code Generation with Iterative Testing and Optimisation.

The first step to this replication process is to replicate the results that the paper has produced for a default model without any use of Agents, and subsequent articles will then elaborate on attempts to replicate and improve performance through the use of Agents. Below can be found the "__main__" method that runs the pipeline for a given model (In this case gpt-3.5-turbo-1106) on a given test set of coding questions we have answers to, and in each section below I will go through what each part of the code is doing.

```python
if __name__ == "__main__":
    model_list = ["gpt-3.5-turbo-1106"]
    language = ["python"]
    print('***** Getting HumanEval dataset *****')
    dataset, construct_few_shot_prompt = get_humaneval_dataset()
    if dataset:
        print('***** HumanEval dataset gotten successfully *****')
        for model in model_list:
            for lg in language:
                updated_dataset = {}
                print(f'***** Running pipeline on {model} for {lg} *****')
                for index, entry in tqdm(enumerate(dataset)):
                    try:
                        updated_entry = fetch_completion(copy.deepcopy(entry), model, lg, construct_few_shot_prompt)
                        updated_dataset[index] = updated_entry
                    except Exception as e:
                        print(repr(e))
                        # If an exception occurs, you may choose to handle it differently, e.g., continue or break the loop.
                        # Here, I'm choosing to skip the current entry and continue to the next one.
                        continue
                with open(f"./dataset/{model}_{lg}.json", "w+") as f:
                    json.dump(updated_dataset, f, indent=4)
    else:
        print('Error: Unable to get HumanEval dataset')
```

## Getting the test data
While the AgentCoder paper has tested multiple models on 2 different evaluations ((HumanEval)[https://huggingface.co/datasets/openai_humaneval] and (MBPP)[https://huggingface.co/datasets/mbpp]) we shall write a reusable implementation and test it against just one to begin.

Both datasets are available via API from Huggingface, and here is some get_humaneval_data function that pulls the first 100 examples:

```python
import requests

'''
# Fetch json dataset from HuggingFace
curl -X GET \
     "https://datasets-server.huggingface.co/rows?dataset=openai_humaneval&config=openai_humaneval&split=test&offset=0&length=100"
'''
def get_humaneval_dataset(): 
    url = "https://datasets-server.huggingface.co/rows"
    params = {
        "dataset": "openai_humaneval",
        "config": "openai_humaneval",
        "split": "test",
        "offset": 0,
        "length": 100
    }

    response = requests.get(url, params=params)

    if response.status_code == 200:
        data = response.json()
        dataset = [entry['row'] for entry in data['rows']]

        construct_few_shot_prompt = f"""
        # For example:

        ## Prompt 1:
        ```python
        {dataset[0]["prompt"]}
        ```

        ## Completion 1:
        ```python
        {dataset[0]["canonical_solution"]}
        ```

        ## Prompt 2:
        ```python
        {dataset[1]["prompt"]}
        ```

        ## Completion 2:
        ```python
        {dataset[1]["canonical_solution"]}
        ```
        """
        return dataset, construct_few_shot_prompt
    else:
        print("Error:", response.status_code)
```

## Getting a benchmark answer

Now that we have downloaded and saved the dataset, we want to get a "completion", or answer from the chosen model for each prompt in the dataset. These prompts are coding questions such as:

```
from typing import List\n\n\ndef has_close_elements(numbers: List[float], threshold: float) -> bool:\n    \"\"\" Check if in given list of numbers, are any two numbers closer to each other than\n    given threshold.\n    >>> has_close_elements([1.0, 2.0, 3.0], 0.5)\n    False\n    >>> has_close_elements([1.0, 2.8, 3.0, 4.0, 5.0, 2.0], 0.3)\n    True\n    \"\"\"\n"
```

After some processing to create an instructional prompt asking the GPT model to generate a solution using the given language, we wrap it in a function that from a given question dataset row, model and language, asks the model to generate a solution N=5 times.

### Function to fetch completion
```python
def fetch_completion(data_entry, model,lg, construct_few_shot_prompt, times = 5):
    if "need_reproduce" in data_entry.keys() and data_entry["need_reproduce"]==False:
        data_entry["time_taken"] = 0
        data_entry["total_tokens"] = 0
        return data_entry #TODO: Why tho???
    
    prompt = data_entry["prompt"]
    text = f"""
        **Role**: You are a software programmer.

        **Task**: As a programmer, you are required to complete the function. Use a Chain-of-Thought approach to break down the problem, create pseudocode, and then write the code in Python language.

        **Code Formatting**: Please write code in ```python\n[Code]\n``` format.

        {construct_few_shot_prompt}

        **Input Code Snippet**:
        ```python
        {prompt}
        ```
        ## Completion 3:
    """

    start_time = time.time()
    completions_code = []
    total_tokens = 0
    for _ in range(times):
        while True:
            try:
                client = OpenAI(
                    # defaults to os.environ.get("OPENAI_API_KEY")
                    api_key=api_key,
                )
                completions = client.chat.completions.create(
                    model=model,
                    stream=False,
                    messages=[
                {"role": "system", "content": "You are a software programmer."},
                {"role": "user", "content":text},
                    ],
                    timeout=100,
                )
                completion = completions.choices[0].message.content
                completion = preprocess_data(completion)
                tokens_used = completions.usage.total_tokens # e.g. "completion_tokens": 17, "prompt_tokens": 57, "total_tokens": 74
                total_tokens += tokens_used
            except Exception as e:
                print(e)
                completion = ""
            if completion!="":
                break
        completions_code.append(completion)
    end_time = time.time()
    elapsed_time = end_time - start_time

    data_entry["completion_list"] = completions_code
    data_entry["time_taken"] = elapsed_time
    data_entry["total_tokens"] = total_tokens
    return data_entry
```

Now that we have a way to get the completions, we can run the function on every line of the question dataset and save the resulting answers for evaluation.

## Evaluating the default responses
 
Having saved the default model responses, let's write some code to see how it's performed on our 100 coding questions.

- Loading Data: The code starts by loading a JSON file containing the dataset. The JSON file is assumed to have a list of dictionaries, each representing a coding task with its associated completion list.
```python
import json
import pandas as pd
import matplotlib.pyplot as plt

with open(r"dataset/downloaded_gpt-3.5-turbo-1106.json", 'r') as f:
    original_data = json.load(f)
    original_data = dict(enumerate(original_data))
```

- Converting to DataFrame: The code converts the loaded data into a Pandas DataFrame for easier manipulation and analysis.
original_df = pd.DataFrame.from_dict(original_data, orient='index')
```python
df = pd.DataFrame.from_dict(original_data, orient='index')
# Convert index to integer
df.index = df.index.astype(int)
df['test_cases'] = df['test'].apply(lambda x: 'def ' + x.split('def ')[-1].strip())
```

- Scorer Function: The scorer function takes a list of example answers and test cases as input. It iterates over each example answer, appends it to the test case, and then attempts to execute the combined code. If any exceptions occur during execution, it prints an error message.
```python
def scorer(example_answers, test_cases):
    temp = []
    for answer in example_answers:
        check = answer.split('def ')
        try:
            func_name = answer.split('def ')[1].split('(')[0].strip()
            code_string = test_cases + '\n\n' + answer + '\n\n' + f'check({func_name})'
            try:
                exec(code_string)
                temp.append(1)
            except Exception:
                temp.append(0)
                print(f'Error with {func_name}')
        except Exception:
            print(f'Error with {check}')
    return temp
```

- get_scores Function: This function iterates over the DataFrame rows, extracts the example answers and test cases for each task, and calculates scores using the scorer function.
```python
def get_scores(df):
    scores = []
    for index, row in df.iterrows():
        example_answers = row['completion_list']
        test_cases = row['test_cases']
        scores.append(scorer(example_answers, test_cases))
    return scores
```

- Calculating Scores: The code then calculates two metrics:
    - The percentage of questions where 
    - The percentage of questions with at least one correct answer.
    - The percentage of all solutions that passed the test cases.
- Printing Results: Finally, the code prints out the calculated percentages.

```python
# Getting performance for the default GPT mode
df['scores'] = scores
df['any_pass'] = [1 if sum(score) >= 1 else 0 for score in scores] # If any succeeded
df['percentage'] = [sum(score)/5 for score in scores] # How many % succeeded
df['one_shot'] = df['scores'].apply(lambda x: x[0]) # How many % succeeded
print('Percentage of questions successfully answered at first pass: ', sum(df['one_shot']), '%')
print('Percentage of questions with at least 1 correct answer: ', sum(df['any_pass']), '%')
print('Percentage of all solutions that passed test cases: ', round(sum(df['percentage']), 2), '%')

# Calculate the percentages
percentage_any_pass = sum(df['any_pass']) / len(df) * 100
percentage_one_shot = sum(df['one_shot']) / len(df) * 100
percentage_overall_pass = sum(df['percentage']) / len(df) * 100

# Plotting
metrics = ['At least one correct answer', 'Successfully answered at first pass', 'Overall solutions passing test cases']
percentages = [percentage_any_pass, percentage_one_shot, percentage_overall_pass]

plt.figure(figsize=(10, 6))
plt.bar(metrics, percentages, color='skyblue')
plt.xlabel('Scoring Metric')
plt.ylabel('Percentage')
plt.title('Performance Metrics')
plt.ylim(0, 100)

# Adding percentages on top of bars
for i, percentage in enumerate(percentages):
    plt.text(i, percentage + 2, f"{percentage:.2f}%", ha='center')

plt.show()
```

From this we can see that our default model actually performs pretty well!

- Percentage of questions successfully answered at first pass:  62 %
- Percentage of questions with at least 1 correct answer:  87 %
- Percentage of all solutions that passed test cases:  63.4 %

![Default Performance](https://notalvin.github.io/posts/images/default_model_performance.png)

### Resource consumption

Beyond purely model performance, we can also measure how much time and tokens were consumed to generate the responses.

Code for plotting performance and token consumption on a question by question basis:

#### How many tokens were consumed to answer each question
```python
# Calculate cumulative changes for overall_pass metric
overall_pass = df['percentage']

# Plotting
fig, ax1 = plt.subplots(figsize=(10, 6))

color = 'tab:blue'
ax1.set_xlabel('Number of Questions')
ax1.set_ylabel('Cumulative Percentage', color=color)
ax1.plot(overall_pass, label='Overall solutions passing test cases', linestyle='-', marker='o', color=color)
ax1.tick_params(axis='y', labelcolor=color)
ax1.set_ylim(-0.1, 1.1)

ax2 = ax1.twinx()  
color = 'tab:red'
ax2.set_ylabel('Tokens Taken', color=color)
ax2.plot(df['total_tokens'], label='Time Taken', linestyle='--', marker='x', color=color)
ax2.tick_params(axis='y', labelcolor=color)

fig.tight_layout()  
fig.legend(loc="lower right")

plt.title('Change in Overall Performance and Tokens Taken')
plt.grid(True)
plt.show()
```
#### How much time was taken to answer each question
```python
ax2 = ax1.twinx()  
color = 'tab:red'
ax2.set_ylabel('Time Taken (seconds)', color=color)
ax2.plot(df['time_taken'], label='Time Taken', linestyle='--', marker='x', color=color)
ax2.tick_params(axis='y', labelcolor=color)

fig.tight_layout()  
fig.legend(loc="lower right")

plt.title('Change in Overall Performance and Time Taken')
plt.grid(True)
plt.show()
```
![Default Performance](https://notalvin.github.io/posts/images/default_model_tokens.png)
![Default Performance](https://notalvin.github.io/posts/images/default_model_time.png)

## Conclusions - Evaluation of default model and next steps
With an overall first pass accuracy of 62%, the default model has performed decently on our 100 question dataset. Based on the above graphs there does not appear to be much correlation between time taken/token consumption and whether the questions were correctly answered. However these metrics will be much more relevant in our next steps, where attempting to use Agents to create a coding workflow will cost much more time and tokens, and we will have to consider if any performance gains are worth considering.

Stay tuned for the next article!