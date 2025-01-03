# Daily Survey Cleaning for TAs

After [imports and loading set up](), to process and analyze daily survey data collected from students, you need to clean the data using the function provided below. 
This function ensures the data is accurate and relevant for analysis by performing the following steps:

## Add the "WeekDay" Column:
Each survey file corresponds to data collected for a specific week and day. The function extracts the `w<week_number>d<day_number>` pattern from filenames and adds it as a new column named WeekDay in the respective DataFrame. This allows you to track the day and week for each entry.

## Combine All DataFrames:
The individual DataFrames for each survey file are merged into a single, comprehensive DataFrame called `combined_df`. This consolidated dataset simplifies and accelerates analysis across all days and weeks.

## Validate Students UID 
To ensure that only matched students for the current year and course are included, the function verifies that all `uid` values in the survey data match the `unique_id` values in the `Students_ReceivedApp_from2021` dataset.

The `Students_ReceivedApp_from2021` dataset contains:

- `course_id:` Identifies the specific course (e.g., course_id = 11).
- `status:` Filters students by application status (e.g., status = "matched").

Rows with uid values not found in the filtered `Students_ReceivedApp_from2021` dataset are removed, ensuring the dataset contains only matched and valied entries for the specified course.

[Please use this document to check the course_id number of interest](https://docs.google.com/document/d/1OUPMUGDOYEmp7Znp4ZrBJSi_-vHU0yBH/edit#heading=h.gjdgxs).

## Remove Duplicate Entries
Duplicate rows based on a combination of WeekDay and uid are identified and removed. Only the first occurrence of each unique combination is retained, ensuring that each student has a single entry per day.

## Save the Cleaned Dataset 
The final cleaned dataset is saved as a CSV file (default: `cleaned_combined_df.csv`).
During this process, the function provides detailed feedback:

- Lists `uid` values in the survey data that are not in the `Students_ReceivedApp_from2021` dataset.
- Reports the number of duplicates identified and removed.


```python
def clean_and_combine_data(dataframes_dict, TAs_apps_path, course_id, status, output_file='combined_df.csv'):
    """
    Cleans and combines dataframes, checks UID validity, handles duplicates, and saves the final combined dataframe.

    Parameters:
    - dataframes_dict: Dictionary with filenames as keys and DataFrames as values.
    - TAs_apps_path: Path to the TAs_apps dataset CSV file.
    - course_id: Course ID to filter the TAs_apps dataset.
    - status: Status to filter the TAs_apps dataset.
    - output_file: Filename to save the cleaned combined dataframe (default: 'combined_df.csv').
    """

    # Step 1: Add "WeekDay" column based on the filename pattern
    for filename, df in dataframes_dict.items():
        match = re.search(r'w\d+d\d+', filename)
        if match:
            df['WeekDay'] = match.group()
        else:
            print(f"No 'WeekDay' pattern found in filename: {filename}")

    # Step 2: Combine all DataFrames
    combined_df = pd.concat(dataframes_dict.values(), ignore_index=True)
    print(f"Combined DataFrame shape: {combined_df.shape}")
    print('#' * 100)
    print(' ')

    # Step 3: Load ST_apps and filter by course_id and status
    ST_apps = pd.read_csv(ST_apps_path)
    ST_apps_filtered = ST_apps[(ST_apps['course_id'] == course_id) & (ST_apps['status'] == status)]
    print(f"Filtered ST_apps shape: {ST_apps_filtered.shape}")
    print('#' * 100)
    print(' ')

    # Step 4: Check that all UID in the new df are present in TA_apps (for the specific course and status that will be specified)
    missing_values = combined_df.loc[~combined_df['uid'].isin(ST_apps_filtered['unique_id']), 'uid']
    if not missing_values.empty:
        print("Values in combined_df['uid'] not found in ST_apps_filtered['unique_id']:", missing_values.tolist())
        combined_df = combined_df[combined_df['uid'].isin(ST_apps_filtered['unique_id'])]
        print("Removed rows with missing UIDs. New shape:", combined_df.shape)
        print('#' * 100)
        print(' ')

    # Step 5: Check for duplicates within each WeekDay and UID combination
    duplicates = combined_df[combined_df.duplicated(subset=['WeekDay', 'uid'], keep=False)]
    if not duplicates.empty:
        print(f"Duplicate rows found based on ['WeekDay', 'uid']: {len(duplicates)}")
        #print(len(duplicates))
        combined_df = combined_df.drop_duplicates(subset=['WeekDay', 'uid'], keep='first')
        print("Removed duplicates. New shape:", combined_df.shape)
        print('#' * 100)
        print(' ')

    # Step 6: Save the cleaned DataFrame to CSV
    combined_df.to_csv(output_file, index=False)
    print(f"Cleaned DataFrame saved to {output_file}")

    return combined_df
```

## Call the function

```python
#Specify the path to the TAs_apps dataset, course ID, and status:
combined_df = clean_and_combine_data(
    dataframes_dict=dataframes_dict,
    ST_apps_path="/content/drive/MyDrive/Academies_DataAnalysis/General/Students_ReceivedApp_from2021.csv", # to be updated in 2025
    course_id=11,       # Specify the course_id number of interest
    status="matched",   # Do NOT change the status
    output_file="cleaned_combined_df.csv" 
)
```