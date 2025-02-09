# TAs Daily Surveys - Cleaning

After [imports and loading set up](), to process and analyze daily survey data collected from TAs (regular, project, and lead), you need to clean the data using the function provided below. 
This function ensures the data is accurate and relevant for analysis by performing the following steps:

## Add the "WeekDay" Column:
Each survey file corresponds to data collected for a specific week and day. The function extracts the `w<week_number>d<day_number>` pattern from filenames and adds it as a new column named WeekDay in the respective DataFrame. This allows you to track the day and week for each entry.

## Combine All DataFrames:
The individual DataFrames for each survey file are merged into a single, comprehensive DataFrame called `combined_df`. This consolidated dataset simplifies and accelerates analysis across all days and weeks.

## Validate TAs Using the TAs_apps Dataset
To ensure that only matched TAs for the current year and course are included, the function verifies that all `uid` values in the survey data match the `unique_id` values in the TAs_apps dataset.

The TAs_apps dataset contains:

- `course_id:` Identifies the specific course (e.g., course_id = 11).
- `status:` Filters TAs by application status (e.g., status = "matched").

Rows with uid values not found in the filtered TAs_apps dataset are removed, ensuring the dataset contains only matched and valied entries for the specified course.

[Please use this document to check the course_id number of interest](https://docs.google.com/document/d/1OUPMUGDOYEmp7Znp4ZrBJSi_-vHU0yBH/edit#heading=h.gjdgxs).

## Remove Duplicate Entries
Duplicate rows based on a combination of WeekDay and uid are identified and removed. Only the first occurrence of each unique combination is retained, ensuring that each TA has a single entry per day.

## Save the Cleaned Dataset 
The final cleaned dataset is saved as a CSV file (default: `cleaned_combined_df.csv`).
During this process, the function provides detailed feedback:

- Lists `uid` values in the survey data that are not in the `TAs_apps` dataset.
- Reports the number of duplicates identified and removed.


```python
def clean_and_combine_data(
    dataframes_dict, 
    TAs_apps_path, 
    course_id, status, 
    ST_apps_path, 
    output_file='combined_df.csv'
):
    """
    Cleans and combines dataframes, checks UID validity, handles duplicates, and saves the final combined dataframe.

    Parameters:
    - dataframes_dict: Dictionary with filenames as keys and DataFrames as values.
    - TAs_apps_path: Path to the TAs_apps dataset CSV file.
    - course_id: Course ID to filter the TAs_apps dataset.
    - status: Status to filter the TAs_apps dataset.
    - ST_apps_path: Path to the ST_apps dataset CSV file.
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
    print('-' * 80)
    print(' ')

    # Step 3: Load TAs_apps and filter by course_id and status
    TAs_apps = pd.read_csv(TAs_apps_path)
    TAs_apps_filtered = TAs_apps[(TAs_apps['course_id'] == course_id) & (TAs_apps['status'] == status)]
    print("[INFO] Filtered TAs_apps dataset for course of interest and TA status = macthed")
    print(f"The shape of this dataset is {TAs_apps_filtered.shape}")
    print('-' * 80)
    print(' ')

    # Step 4: Check that all UID in the new df are present in TA_apps (for the specific course and status that will be specified)
    missing_values = combined_df.loc[~combined_df['uid'].isin(TAs_apps_filtered['unique_id']), 'uid']
    if not missing_values.empty:
        # print(f"{len(missing_values)} are not found in the TA_apps dataset filtered for matched and course_id")
        print(f"[WARNING]: {missing_values.nunique()} unique_id are not found in the TA_apps dataset filtered for matched and course_id")
        print("List:", missing_values.tolist())
        print('-' * 80)
        print(' ')

        # 4A: Check these missing UIDs in the FULL ST_apps
        ST_apps = pd.read_csv(ST_apps_path)
        print("[INFO] Checking the students dataset for these missing UIDs...")
        print("       They may exist under different statuses or different courses")
        missing_in_ST_full = ST_apps[ST_apps['unique_id'].isin(missing_values)]
        found_uids_ST = missing_in_ST_full['unique_id'].unique()

        if found_uids_ST.size > 0:
            print("[INFO] Some of the missing UIDs were found in the students dataset:")
            for uid in found_uids_ST:
                uid_rows = missing_in_ST_full.loc[
                    missing_in_ST_full['unique_id'] == uid,
                    ['application_status', 'course_id']
                ]
                statuses = uid_rows['application_status'].unique()
                course_ids = uid_rows['course_id'].unique()
                print(f"  UID = {uid}, status = {list(statuses)}, course_id = {list(course_ids)}")

        # Count the number of rows before filtering
        original_count = combined_df.shape[0]

        # Count the number of rows that will remain after filtering
        remaining_count = combined_df[combined_df['uid'].isin(TAs_apps_filtered['unique_id'])].shape[0]

        # Calculate the number of rows that will be removed
        removed_count = original_count - remaining_count
        print('-' * 80)
        print(f"Number of rows to be removed: {removed_count}")
        print('-' * 80)
        print(' ')
        combined_df = combined_df[combined_df['uid'].isin(TAs_apps_filtered['unique_id'])]
        print("Removed rows with missing UIDs. New shape:", combined_df.shape)
        print('-' * 80)
        print(' ')

    # Step 5: Check for duplicates within each WeekDay and UID combination
    duplicates = combined_df[combined_df.duplicated(subset=['WeekDay', 'uid'], keep=False)]
    if not duplicates.empty:
        print(f"Duplicate rows found based on ['WeekDay', 'uid']: {len(duplicates)}")
        #print(len(duplicates))
        combined_df = combined_df.drop_duplicates(subset=['WeekDay', 'uid'], keep='first')
        print("Removed duplicates. New shape:", combined_df.shape)
        print('-' * 80)
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
    dataframes_dict =dataframes_dict,
    TAs_apps_path   ="/content/drive/MyDrive/Academies_DataAnalysis/General/TAs_ReceivedApp_from2021.csv", # to be updated in 2025
    course_id       =12,       # Specify the course_id number of interest
    status          ="matched",   # Do NOT change the status
    ST_apps_path    ="/content/drive/MyDrive/Academies_DataAnalysis/General/Students_ReceivedApp_from2021.csv", # to be updated in 2025
    output_file     ="cleaned_combined_df.csv" 
)
```