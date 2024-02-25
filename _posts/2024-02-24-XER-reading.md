---
title: Parsing XER files data
date: 2024-02-24
---

![Structure](/home/docs/assets/2024-02-24-XER-reading/pexels-ave-calvar-martinez-5610090.jpg)

This post is an extension of the previous one ["Optimizing Project Planning and Material transportation"](https://inigmat.github.io/home/2023/11/29/Transport-and-Resources.html). The previous post describes some techniques how for optimizing the schedule, where the data is prepared manually. So, the current article will show the method of obtaining the data from Primavera P6 project files (*.xer).

Looking for libraries in Python programming language, I have found two:
- [xerparser](https://pypi.org/project/xerparser/)  
- [PyP6Xer](https://pypi.org/project/PyP6Xer/).

Both look to have the same features at first glance. The first one is chosen for this case.

The content below is based on the ipython notebook, so there is an option to check it out on [nbviewer.org](https://nbviewer.org/github/inigmat/exupery/blob/main/XER_reading.ipynb).

# Intro

This notebook demonstrates an example of how to use [the Primavera P6 xer files reader](https://pypi.org/project/xerparser/0.10.3/) library for data preparation.

The XER file contains the same data presented in the [notebook](https://nbviewer.org/github/inigmat/exupery/blob/main/Schedule_CPLEX.ipynb).

By obtaining the file via URL link, the required data—task names, durations, precedences, and release dates (intervals between each house's earliest starting date)—is transformed into Python objects.


# Installing xerparser



```python
import importlib

# Check if xerparser is already installed
try:
    importlib.import_module('xerparser')
    print("xerparser is already installed.")
except ImportError:
    try:
        # Attempt installation
        %pip install xerparser
    except Exception as e:
        print("An error occurred while installing xerparser:", e)
```


# Getting the XER file


```python
import requests

url = "https://raw.githubusercontent.com/inigmat/exupery/main/files/schedule/MDL4D.xer"

try:
    response = requests.get(url)
    if response.status_code == 200:
        print("Request successful!")
    else:
        print(f"Failed to retrieve data. Status code: {response.status_code}")
except requests.exceptions.RequestException as e:
    print(f"An error occurred: {e}")
```


# Read the downloaded file with checking for errors


```python
from xerparser import Xer, CorruptXerFile

try:
    xer = Xer(response.text)
except CorruptXerFile as e:
    for error in e.errors:
        print(error)
```

# Getting the project data


```python
project = list(xer.projects.values())[0]
```

We have to get the following data

```
NbWorkers = 3
NbHouses  = 5

TaskNames = ("masonry","carpentry","plumbing",
             "ceiling","roofing","painting",
             "windows","facade","garden","moving")

Duration =  [35, 15, 40, 15, 5, 10, 5, 10, 5, 5]

ReleaseDate = [31, 0, 90, 120, 90]

Precedences = [("masonry", "carpentry"), ("masonry", "plumbing"), ("masonry", "ceiling"),
               ("carpentry", "roofing"), ("ceiling", "painting"), ("roofing", "windows"),
               ("roofing", "facade"), ("plumbing", "facade"), ("roofing", "garden"),
               ("plumbing", "garden"), ("windows", "moving"), ("facade", "moving"),
               ("garden", "moving"), ("painting", "moving")]
```



# Start with getting the WBS (house numbers)

We skip the first two levels using WBS_LVL variable


```python
WBS_LVL = 2
houses = [obj.name for obj in project.wbs_nodes[WBS_LVL:]]
NbHouses= len(houses)
```


# Obtain the labor resources of the project to get the number of workers


```python
workers = {}
for res in project.resources:
  if res.rsrc_type == "RT_Labor":
    workers[res.rsrc_id] = res.resource.name
NbWorkers = len(workers)
```

# Prepare pandas dataframe to check out the tasks data such as 'TASK ID','Name','Type', 'WBS ID', 'Duration', 'Successors (ID, Link, Lag)'


```python
import pandas as pd

tasks_df = pd.DataFrame(columns=['TASK ID','Name','Type', 'WBS ID', 'Duration', 'Successors (ID, Link, Lag)'])

for task in project.tasks:
    task_sucs = []
    for pred_link in task.successors:
        task_suc = pred_link.task.uid
        task_link = pred_link.link
        task_lag = pred_link.lag
        task_sucs.append((task_suc, task_link, task_lag))

    tasks_df = pd.concat([tasks_df, pd.DataFrame(
        {'TASK ID': [task.uid],
         'Name': [task.name],
         'Type': [task.type],
         'WBS ID': [task.wbs_id],
         'Duration': [task.duration],
         'Successors (ID, Link, Lag)': [task_sucs],
         })], ignore_index=True)
```


```python
tasks_df.head(15)
```
Output:

  <div id="df-11e5c0b1-13df-42bc-980f-216548301c26" class="colab-df-container">
    <div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>TASK ID</th>
      <th>Name</th>
      <th>Type</th>
      <th>WBS ID</th>
      <th>Duration</th>
      <th>Successors (ID, Link, Lag)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>104836</td>
      <td>House 1 Start</td>
      <td>TaskType.TT_Mile</td>
      <td>26152</td>
      <td>0</td>
      <td>[]</td>
    </tr>
    <tr>
      <th>1</th>
      <td>104837</td>
      <td>Masonry</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>35</td>
      <td>[(104838, FS, 0), (104839, FS, 0), (104840, FS...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>104838</td>
      <td>Carpentry</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>15</td>
      <td>[(104841, FS, 0)]</td>
    </tr>
    <tr>
      <th>3</th>
      <td>104839</td>
      <td>Plumbing</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>40</td>
      <td>[(104844, FS, 0), (104845, FS, 0)]</td>
    </tr>
    <tr>
      <th>4</th>
      <td>104840</td>
      <td>Ceiling</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>15</td>
      <td>[(104842, FS, 0)]</td>
    </tr>
    <tr>
      <th>5</th>
      <td>104841</td>
      <td>Roofing</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>5</td>
      <td>[(104843, FS, 0), (104844, FS, 0), (104845, FS...</td>
    </tr>
    <tr>
      <th>6</th>
      <td>104842</td>
      <td>Painting</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>10</td>
      <td>[(104846, FS, 0)]</td>
    </tr>
    <tr>
      <th>7</th>
      <td>104843</td>
      <td>Windows</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>5</td>
      <td>[(104846, FS, 0)]</td>
    </tr>
    <tr>
      <th>8</th>
      <td>104844</td>
      <td>Facade</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>10</td>
      <td>[(104846, FS, 0)]</td>
    </tr>
    <tr>
      <th>9</th>
      <td>104845</td>
      <td>Garden</td>
      <td>TaskType.TT_Task</td>
      <td>26154</td>
      <td>5</td>
      <td>[(104846, FS, 0)]</td>
    </tr>
    <tr>
      <th>10</th>
      <td>104846</td>
      <td>Moving</td>
      <td>TaskType.TT_Rsrc</td>
      <td>26154</td>
      <td>5</td>
      <td>[(104883, FF, 0)]</td>
    </tr>
    <tr>
      <th>11</th>
      <td>104847</td>
      <td>House 2 Start</td>
      <td>TaskType.TT_Mile</td>
      <td>26152</td>
      <td>0</td>
      <td>[]</td>
    </tr>
    <tr>
      <th>12</th>
      <td>104848</td>
      <td>Masonry</td>
      <td>TaskType.TT_Task</td>
      <td>26155</td>
      <td>35</td>
      <td>[(104849, FS, 0), (104850, FS, 0), (104851, FS...</td>
    </tr>
    <tr>
      <th>13</th>
      <td>104849</td>
      <td>Carpentry</td>
      <td>TaskType.TT_Task</td>
      <td>26155</td>
      <td>15</td>
      <td>[(104852, FS, 0)]</td>
    </tr>
    <tr>
      <th>14</th>
      <td>104850</td>
      <td>Plumbing</td>
      <td>TaskType.TT_Task</td>
      <td>26155</td>
      <td>40</td>
      <td>[(104855, FS, 0), (104856, FS, 0)]</td>
    </tr>
  </tbody>
</table>
</div>
    <div class="colab-df-buttons">

  <div class="colab-df-container">
    <button class="colab-df-convert" onclick="convertToInteractive('df-11e5c0b1-13df-42bc-980f-216548301c26')"
            title="Convert this dataframe to an interactive table."
            style="display:none;">

  <svg xmlns="http://www.w3.org/2000/svg" height="24px" viewBox="0 -960 960 960">
    <path d="M120-120v-720h720v720H120Zm60-500h600v-160H180v160Zm220 220h160v-160H400v160Zm0 220h160v-160H400v160ZM180-400h160v-160H180v160Zm440 0h160v-160H620v160ZM180-180h160v-160H180v160Zm440 0h160v-160H620v160Z"/>
  </svg>
    </button>

  <style>
    .colab-df-container {
      display:flex;
      gap: 12px;
    }

    .colab-df-convert {
      background-color: #E8F0FE;
      border: none;
      border-radius: 50%;
      cursor: pointer;
      display: none;
      fill: #1967D2;
      height: 32px;
      padding: 0 0 0 0;
      width: 32px;
    }

    .colab-df-convert:hover {
      background-color: #E2EBFA;
      box-shadow: 0px 1px 2px rgba(60, 64, 67, 0.3), 0px 1px 3px 1px rgba(60, 64, 67, 0.15);
      fill: #174EA6;
    }

    .colab-df-buttons div {
      margin-bottom: 4px;
    }

    [theme=dark] .colab-df-convert {
      background-color: #3B4455;
      fill: #D2E3FC;
    }

    [theme=dark] .colab-df-convert:hover {
      background-color: #434B5C;
      box-shadow: 0px 1px 3px 1px rgba(0, 0, 0, 0.15);
      filter: drop-shadow(0px 1px 2px rgba(0, 0, 0, 0.3));
      fill: #FFFFFF;
    }
  </style>

    <script>
      const buttonEl =
        document.querySelector('#df-11e5c0b1-13df-42bc-980f-216548301c26 button.colab-df-convert');
      buttonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';

      async function convertToInteractive(key) {
        const element = document.querySelector('#df-11e5c0b1-13df-42bc-980f-216548301c26');
        const dataTable =
          await google.colab.kernel.invokeFunction('convertToInteractive',
                                                    [key], {});
        if (!dataTable) return;

        const docLinkHtml = 'Like what you see? Visit the ' +
          '<a target="_blank" href=https://colab.research.google.com/notebooks/data_table.ipynb>data table notebook</a>'
          + ' to learn more about interactive tables.';
        element.innerHTML = '';
        dataTable['output_type'] = 'display_data';
        await google.colab.output.renderOutput(dataTable, element);
        const docLink = document.createElement('div');
        docLink.innerHTML = docLinkHtml;
        element.appendChild(docLink);
      }
    </script>
  </div>


<div id="df-f88aaf74-ee16-4e74-a1c1-3d3afd25655b">
  <button class="colab-df-quickchart" onclick="quickchart('df-f88aaf74-ee16-4e74-a1c1-3d3afd25655b')"
            title="Suggest charts"
            style="display:none;">

<svg xmlns="http://www.w3.org/2000/svg" height="24px"viewBox="0 0 24 24"
     width="24px">
    <g>
        <path d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zM9 17H7v-7h2v7zm4 0h-2V7h2v10zm4 0h-2v-4h2v4z"/>
    </g>
</svg>
  </button>

<style>
  .colab-df-quickchart {
      --bg-color: #E8F0FE;
      --fill-color: #1967D2;
      --hover-bg-color: #E2EBFA;
      --hover-fill-color: #174EA6;
      --disabled-fill-color: #AAA;
      --disabled-bg-color: #DDD;
  }

  [theme=dark] .colab-df-quickchart {
      --bg-color: #3B4455;
      --fill-color: #D2E3FC;
      --hover-bg-color: #434B5C;
      --hover-fill-color: #FFFFFF;
      --disabled-bg-color: #3B4455;
      --disabled-fill-color: #666;
  }

  .colab-df-quickchart {
    background-color: var(--bg-color);
    border: none;
    border-radius: 50%;
    cursor: pointer;
    display: none;
    fill: var(--fill-color);
    height: 32px;
    padding: 0;
    width: 32px;
  }

  .colab-df-quickchart:hover {
    background-color: var(--hover-bg-color);
    box-shadow: 0 1px 2px rgba(60, 64, 67, 0.3), 0 1px 3px 1px rgba(60, 64, 67, 0.15);
    fill: var(--button-hover-fill-color);
  }

  .colab-df-quickchart-complete:disabled,
  .colab-df-quickchart-complete:disabled:hover {
    background-color: var(--disabled-bg-color);
    fill: var(--disabled-fill-color);
    box-shadow: none;
  }

  .colab-df-spinner {
    border: 2px solid var(--fill-color);
    border-color: transparent;
    border-bottom-color: var(--fill-color);
    animation:
      spin 1s steps(1) infinite;
  }

  @keyframes spin {
    0% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
      border-left-color: var(--fill-color);
    }
    20% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    30% {
      border-color: transparent;
      border-left-color: var(--fill-color);
      border-top-color: var(--fill-color);
      border-right-color: var(--fill-color);
    }
    40% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-top-color: var(--fill-color);
    }
    60% {
      border-color: transparent;
      border-right-color: var(--fill-color);
    }
    80% {
      border-color: transparent;
      border-right-color: var(--fill-color);
      border-bottom-color: var(--fill-color);
    }
    90% {
      border-color: transparent;
      border-bottom-color: var(--fill-color);
    }
  }
</style>

  <script>
    async function quickchart(key) {
      const quickchartButtonEl =
        document.querySelector('#' + key + ' button');
      quickchartButtonEl.disabled = true;  // To prevent multiple clicks.
      quickchartButtonEl.classList.add('colab-df-spinner');
      try {
        const charts = await google.colab.kernel.invokeFunction(
            'suggestCharts', [key], {});
      } catch (error) {
        console.error('Error during call to suggestCharts:', error);
      }
      quickchartButtonEl.classList.remove('colab-df-spinner');
      quickchartButtonEl.classList.add('colab-df-quickchart-complete');
    }
    (() => {
      let quickchartButtonEl =
        document.querySelector('#df-f88aaf74-ee16-4e74-a1c1-3d3afd25655b button');
      quickchartButtonEl.style.display =
        google.colab.kernel.accessAllowed ? 'block' : 'none';
    })();
  </script>
</div>
    </div>
  </div>




# Get the list of task names. Filter by wbs_id due to the repeating structure of activities on each house


```python
def get_task_dict_by_wbs_id(df, wbs_id):
    filtered_df = df[df['WBS ID'] == wbs_id]
    task_dict = dict(zip(filtered_df['TASK ID'], filtered_df['Name']))
    return task_dict

wbs_id = '26154' # wbs_id of activites on House 1 (taken from the dataframe above)
task_dict = get_task_dict_by_wbs_id(tasks_df, wbs_id)
TaskNames = tuple(task_dict.values())
```

# Get tasks duration


```python
durations = {task_id: tasks_df.loc[tasks_df['TASK ID'] == task_id, 'Duration'].values[0] for task_id in task_dict.keys()}
Duration =  list(durations.values())
```

# Get the precedences of the tasks


```python
prec = []
for task in project.tasks:
  if task.uid in task_dict:
    for pred_link in task.successors:
      suc_uid = pred_link.task.uid
      if suc_uid in task_dict:
        suc_name = pred_link.task.name
        prec.append((task.name,suc_name))
Precedences = prec
```

# Get the release dates. Dates ordered by the house number


```python
lags = {}
for rel in project.relationships:
    if str(rel.predecessor.type) == 'TaskType.TT_Mile':
      lags[rel.successor.wbs.name] = rel.lag
```


```python
ReleaseDate_dict = {key: lags.get(key) for key in sorted(lags)}
ReleaseDate = list(ReleaseDate_dict.values())
```

# Finally obtain the project data


```python
print("NbWorkers =", NbWorkers, "\n")
print("NbHouses =", NbHouses, "\n")
print("TaskNames =", TaskNames, "\n")
print("Duration =", Duration, "\n")
print("ReleaseDate =", ReleaseDate, "\n")
print("Precedences =", Precedences, "\n")
```

Output:

    NbWorkers = 3 
    
    NbHouses = 5 
    
    TaskNames = ('Masonry', 'Carpentry', 'Plumbing', 'Ceiling', 'Roofing', 'Painting', 'Windows', 'Facade', 'Garden', 'Moving') 
    
    Duration = [35, 15, 40, 15, 5, 10, 5, 10, 5, 5] 
    
    ReleaseDate = [31, 0, 90, 120, 90] 
    
    Precedences = [('Masonry', 'Carpentry'), ('Masonry', 'Plumbing'), ('Masonry', 'Ceiling'), ('Carpentry', 'Roofing'), ('Plumbing', 'Facade'), ('Plumbing', 'Garden'), ('Ceiling', 'Painting'), ('Roofing', 'Windows'), ('Roofing', 'Facade'), ('Roofing', 'Garden'), ('Painting', 'Moving'), ('Windows', 'Moving'), ('Facade', 'Moving'), ('Garden', 'Moving')] 
