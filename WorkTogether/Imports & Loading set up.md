# Imports set up

---
All datasets should be cleaned in the same way and throught the notebook you will see how the cleaning function will be shaped for cleaning daily surveys for TAs, students, enrolment and post course survey. Please follow the pipelines created. 

---
At the top of every notebook, we define an **Import section** where all necessary packages and libraries should be imported. This ensures that all dependencies are loaded at the beginning of the notebook, making it easy to track what packages are in use.

Below are some recommended packages and configurations that enhance data analysis and visualization quality:

```python
# General imports
import pandas as pd  # For data manipulation and analysis
import numpy as np   # For numerical operations
import glob          # For file searching
import re            # For pattern matching

# Visualization imports
import matplotlib.pyplot as plt  # Basic plotting
import seaborn as sns            # Statistical data visualization

# Display high-quality plots in SVG format
from IPython import display
display.set_matplotlib_formats('svg')  # Recommended for better plot quality

# to restrict the float value to 2 decimal places
pd.set_option('display.float_format', lambda x: '%.2f' % x)

# display max numer of columns. Change the value as needed.
pd.set_option('display.max_columns', 100)

```
# Loading set up

**Data Loading** will be our second section. We use Google Drive to load datasets. 

The code below mounts Google Drive, finds all CSV files with a specified prefix in a given folder, and loads each CSV file into a dictionary of DataFrames for easy access.

Instructions:

1. Mount Google Drive to access files directly from your Drive.
```python
from google.colab import drive, files
drive.mount('/content/drive')
```

2. Specify the Folder Path and File Prefix: Set the folder path where the CSV files are stored and the prefix for the files you want to load (e.g., CS for files starting with "CS").
```python
folder_path = "/content/drive/MyDrive/Academies_DataAnalysis/July2024/DailySurveys/CMA/TAs/"
file_prefix = "CS"  # Replace with your desired prefix
```

3. Load Files Using glob: The code uses glob.glob function to identify all CSV files matching the prefix and loads them into a dictionary, with each filename (minus the path) as the key.
```python
csv_files       = glob.glob(f"{folder_path}/{file_prefix}*.csv")
dataframes_dict = {file.split('/')[-1]: pd.read_csv(file) for file in csv_files}

# To access each DataFrame by filename, use the dictionary
# Example: Accessing the 'CSw1d1-Grid view.csv' file if it exists
dataframe_name = 'CSw1d1-Grid view.csv' # Adjust the filename to match your data
if dataframe_name in dataframes_dict:
    df = dataframes_dict[dataframe_name]
    print(df.head())  # Display the first 5 rows 
else:
    print(f"{dataframe_name} not found in the loaded files.")

# Print the loaded dataframes
print("Loaded DataFrames:", dataframes_dict.keys())

# Check the number of datasets loaded
len(dataframes_dict)
```

