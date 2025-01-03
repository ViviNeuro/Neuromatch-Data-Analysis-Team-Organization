# Students Daily Surveys - Cleaning

After [imports and loading set up](https://vivineuro.github.io/Neuromatch-Data-Analysis-Team-Organization/WorkTogether/Imports%20%26%20Loading%20set%20up.html), to process and analyze daily survey data collected from students, you need to clean the data using the function provided below. 
This function ensures the data is accurate and relevant for analysis by performing the following steps:

## Add the "WeekDay" Column:
Each survey file corresponds to data collected for a specific week and day. The function extracts the `w<week_number>d<day_number>` pattern from filenames and adds it as a new column named WeekDay in the respective DataFrame. This allows you to track the day and week for each entry.

## Combine All DataFrames:
The individual DataFrames for each survey file are merged into a single, comprehensive DataFrame called `combined_df`. This consolidated dataset simplifies and accelerates analysis across all days and weeks.

## Validate Students UID 
To ensure that only matched students for the current year and course are included, the function verifies that all `uid` values in the survey data match the `unique_id` values in the `Students_ReceivedApp_from2021`.

The `Students_ReceivedApp_from2021` dataset contains:

- `course_id:` Identifies the specific course (e.g., course_id = 11).
- `application_status:` Filters students by application status (e.g., status = "matched").


Rows with uid values not found in the filtered `Students_ReceivedApp_from2021` dataset are first cross-referenced with the complete (unfiltered) `Students_ReceivedApp_from2021` dataset to undertand where are these uid coming from. 

If there are still unmatched uid values, they are then checked against the TAs dataset.

After these verifications, the function retains only the uid values that are matched and are valid entries for the specified course, removing all others.


[Please use this document to check the course_id number of interest](https://docs.google.com/document/d/1OUPMUGDOYEmp7Znp4ZrBJSi_-vHU0yBH/edit#heading=h.gjdgxs).

## Remove Duplicate Entries
Duplicate rows based on a combination of WeekDay and uid are identified and removed. Only the first occurrence of each unique combination is retained, ensuring that each student has a single entry per day.

## Save the Cleaned Dataset 
The final cleaned dataset is saved as a CSV file (default: `cleaned_combined_df.csv`).
During this process, the function provides detailed feedback:

- Lists `uid` values in the survey data that are not in the `Students_ReceivedApp_from2021` dataset.
- Reports the number of duplicates identified and removed.


```python
def clean_and_combine_data(
    dataframes_dict,
    ST_apps_path,
    course_id,
    status,
    TAs_apps_path,
    output_file='combined_df.csv'
):
    """
    Cleans and combines dataframes, checks UID validity, handles duplicates,
    and saves the final combined dataframe.

    Parameters:
    - dataframes_dict: Dictionary with filenames as keys and DataFrames as values.
    - ST_apps_path: Path to the ST_apps dataset CSV file.
    - course_id: Course ID to filter the ST_apps dataset.
    - status: Status to filter the ST_apps dataset.
    - TAs_apps_path: Path to the TAs_apps dataset CSV file.
    - output_file: Filename to save the cleaned combined dataframe (default: 'combined_df.csv').
    """

    # Step 1: Add "WeekDay" column based on the filename pattern
    for filename, df in dataframes_dict.items():
        match = re.search(r'w\d+d\d+', filename, re.IGNORECASE)
        if match:
            df['WeekDay'] = match.group()
        else:
            print(f"No 'WeekDay' pattern found in filename: {filename}")

    # Step 2: Combine all DataFrames
    combined_df = pd.concat(dataframes_dict.values(), ignore_index=True)
    print(f"Combined DataFrame shape: {combined_df.shape}")
    print('#' * 100)
    print()

    # Step 3: Load ST_apps (full) and TAs_apps, then filter ST_apps by course_id and status
    ST_apps = pd.read_csv(ST_apps_path)
    TAs_apps = pd.read_csv(TAs_apps_path)

    ST_apps_filtered = ST_apps[
        (ST_apps['course_id'] == course_id) &
        (ST_apps['application_status'] == status)
    ]
    print(f"Filtered ST_apps shape: {ST_apps_filtered.shape}")
    print('#' * 100)
    print()

    # Step 4: Identify UIDs in combined_df that are NOT in ST_apps_filtered
    missing_values = combined_df.loc[
        ~combined_df['uid'].isin(ST_apps_filtered['unique_id']), 'uid'
    ].unique()

    if missing_values.size > 0:
        print(f"There are {len(missing_values)} values in combined_df['uid'] NOT found in ST_apps_filtered['unique_id']:")
        print(missing_values)

        # 4A: Check these missing UIDs in the FULL ST_apps
        missing_in_ST_full = ST_apps[ST_apps['unique_id'].isin(missing_values)]
        found_uids_ST = missing_in_ST_full['unique_id'].unique()

        if found_uids_ST.size > 0:
            print("\n—> Missing UIDs found in the FULL ST_apps:")
            for uid in found_uids_ST:
                uid_rows = missing_in_ST_full.loc[
                    missing_in_ST_full['unique_id'] == uid,
                    ['application_status', 'course_id']
                ]
                statuses = uid_rows['application_status'].unique()
                course_ids = uid_rows['course_id'].unique()
                print(f"  UID = {uid}, status = {list(statuses)}, course_id = {list(course_ids)}")

        # 4B: After ST check, see which UIDs remain missing
        still_missing_after_ST = set(missing_values) - set(found_uids_ST)
        if len(still_missing_after_ST) > 0:
            print(f"\n—> These {len(still_missing_after_ST)} UIDs are still missing after checking FULL ST_apps:")
            print(still_missing_after_ST)

            # Now check the still-missing UIDs in TAs_apps
            missing_in_TAs = TAs_apps[TAs_apps['unique_id'].isin(still_missing_after_ST)]
            found_uids_TAs = missing_in_TAs['unique_id'].unique()

            if found_uids_TAs.size > 0:
                print("\n—> Missing UIDs found in the TAs_apps dataset:")
                for uid in found_uids_TAs:
                    uid_rows = missing_in_TAs.loc[
                        missing_in_TAs['unique_id'] == uid,
                        ['status', 'course_id']
                    ]
                    statuses = uid_rows['status'].unique()
                    course_ids = uid_rows['course_id'].unique()
                    print(f"  UID = {uid}, status = {list(statuses)}, course_id = {list(course_ids)}")

            # UIDs not found anywhere (neither in full ST nor TAs)
            not_found_anywhere = still_missing_after_ST - set(found_uids_TAs)
            if not_found_anywhere:
                print(f"\n—> The following missing {len(not_found_anywhere)} UIDs were NOT found in ST_apps_filtered, FULL ST_apps, or TAs_apps:")
                print(not_found_anywhere)
        else:
            # If nothing left after the ST check, skip TAs_apps checks
            found_uids_TAs = []

        # Finally, remove rows from combined_df that are NOT in ST_apps_filtered
        combined_df = combined_df[combined_df['uid'].isin(ST_apps_filtered['unique_id'])]
        print("\nRemoved rows with missing UIDs that are not in the filtered list.")
        print("New shape:", combined_df.shape)
        print('#' * 100)
        print()

    # Step 5: Check for duplicates within each WeekDay and UID combination
    duplicates = combined_df[combined_df.duplicated(subset=['WeekDay', 'uid'], keep=False)]
    if not duplicates.empty:
        print(f"Duplicate rows found based on ['WeekDay', 'uid']: {len(duplicates)}")
        combined_df = combined_df.drop_duplicates(subset=['WeekDay', 'uid'], keep='first')
        print("Removed duplicates. New shape:", combined_df.shape)
        print('#' * 100)
        print()

    # Step 6: Save the cleaned DataFrame to CSV
    combined_df.to_csv(output_file, index=False)
    print(f"Cleaned DataFrame saved to {output_file}")

    return combined_df
```



```python
combined_df = clean_and_combine_data(
    dataframes_dict=dataframes_dict,
    ST_apps_path="/content/drive/MyDrive/Academies_DataAnalysis/General/Students_ReceivedApp_from2021.csv", # to be updated in 2025
    course_id=11,       # Specify the course_id number of interest
    status="matched",   # Do NOT change the status
    TAs_apps_path = "/content/drive/MyDrive/Academies_DataAnalysis/General/TAs_ReceivedApp_from2021.csv",
    output_file="cleaned_combined_df.csv" 
)
```