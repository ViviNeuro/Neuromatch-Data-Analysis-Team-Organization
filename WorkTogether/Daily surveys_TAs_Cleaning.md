# TAs Daily Surveys - Cleaning

After [imports and loading set up](), to process and analyze daily survey data collected from TAs (regular, project, and lead), you need to clean the data using the function provided below. 
This function ensures the data is accurate and relevant for analysis by performing the following processing steps:

## 1. Extracting WeekDay from Filenames:
For each DataFrame in the `dataframes_dict`, the function uses a regular expression (`r'w\d+d\d+'`) to extract a pattern (representing "WeekDay") from the filename. This pattern is added as a new column (`WeekDay`). If the pattern is not found, a warning is printed.

## 2. Combining DataFrames:
All DataFrames from the dictionary are concatenated into a single DataFrame (`combined_df`), and its shape is printed.

## 3. Filtering TA Applications:

- Loads the TAs applications CSV file using the provided `TAs_apps_path`.
- Filters the dataset for rows that match the specified `course_id` and `status`.
- Prints the shape of the filtered TA dataset.

## 4. Processing Certificates:

- Loads the certificates CSV file using the provided `certificates_path`.
- Filters for rows that match the given `course_id` and have a certificate type within the set ['ta_certificate', 'project_ta_certificate', 'lead_ta_certificate'].
- Maps each certificate type to a TA role (using a predefined mapping) and adds a new column (`TA_role`).
- Creates mapping series (based on `unique_id`) for both TA_role and certificate_type.
- Maps these values to the filtered TAs applications DataFrame.

## 5. UID Validation:

- Checks that every `uid` in `combined_df` exists in the filtered TAs applications dataset.
- If missing UIDs are found, it reports these UIDs and checks them against the full student applications dataset (using `ST_apps_path`), printing any additional details.
- Filters out rows in `combined_df` whose UIDs are not present in the filtered TAs applications.

## 6. Merging TA Information:
Merges the filtered TA applications DataFrame (with `TA role`, `certificate type`, and `career_status` columns) into the combined DataFrame on `uid` (from combined_df) and `unique_id` (from the TA dataset).

## 7. Removing Duplicate Entries:
Checks for duplicate entries based on the combination of `WeekDay` and `uid`. If duplicates are found, only the first occurrence is kept.

## 8. Saving the Output:
The final cleaned DataFrame is saved to a CSV file with the name specified by the `output_file` parameter. A confirmation message is printed.



```python
def clean_and_combine_data(
    dataframes_dict,
    TAs_apps_path,
    course_id,
    status,
    ST_apps_path,
    certificates_path,
    output_file='combined_df.csv'):
  
    """
    Cleans and combines dataframes, checks UID validity, handles duplicates, and saves the final combined dataframe.
    Additionally, loads the certificates dataset, adds the TA_role and certificate received based on matching UIDs.

    Parameters:
    - dataframes_dict: Dictionary with filenames as keys and DataFrames as values.
    - TAs_apps_path: Path to the TAs_apps dataset CSV file.
    - course_id: Course ID to filter the TAs_apps dataset.
    - status: Status to filter the TAs_apps dataset.
    - ST_apps_path: Path to the ST_apps dataset CSV file.
    - certificates_path: Path to the certificates dataset CSV file.
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
    TAs_apps_filtered = TAs_apps[(TAs_apps['course_id'] == course_id) & (TAs_apps['status'] == status)].copy()
    print("[INFO] Filtered TAs_apps dataset for course of interest and TA status = matched")
    print(f"The shape of this dataset is {TAs_apps_filtered.shape}")
    print('-' * 80)
    print(' ')

    # Load certificates dataset
    certificates = pd.read_csv(certificates_path)

    # Filter certificates for course_id and certificate type
    certificates = certificates[(certificates['course_id'] == course_id) & (certificates['certificate_type'].isin(['ta_certificate', 'project_ta_certificate', 'lead_ta_certificate']))]
    print(f"[INFO] Filtered certificates dataset for course_id = {course_id} and certificate_type")
    print(f"Certificates shape: {certificates.shape}")
    print('-' * 80)
    print(' ')

    # Create a new column 'TA_role' based on certificate_type values
    mapping = {
        'ta_certificate': 'Regular TA',
        'project_ta_certificate': 'Project TA',
        'lead_ta_certificate': 'Lead TA'}

    certificates['TA_role'] = certificates['certificate_type'].map(mapping)

    # Create mapping series for TA_role and certificate_type using unique_id as index
    TA_role_mapping = certificates.set_index('unique_id')['TA_role']
    certificate_type_mapping = certificates.set_index('unique_id')['certificate_type']

    # Map the values to TAs_apps_filtered based on unique_id
    TAs_apps_filtered['TA_role'] = TAs_apps_filtered['unique_id'].map(TA_role_mapping)
    TAs_apps_filtered['certificate_type'] = TAs_apps_filtered['unique_id'].map(certificate_type_mapping)

    # Step 4: Check that all UID in the combined dataframe are present in TAs_apps_filtered
    missing_values = combined_df.loc[~combined_df['uid'].isin(TAs_apps_filtered['unique_id']), 'uid']
    if not missing_values.empty:
        print(f"[WARNING]: {missing_values.nunique()} unique_id are not found in the TA_apps dataset filtered for matched and course_id")
        print("List:", missing_values.tolist())
        print('-' * 80)
        print(' ')
        
        # 4A: Check these missing UIDs in the FULL ST_apps
        ST_apps = pd.read_csv(ST_apps_path)
        print("[INFO] Checking whether the unique_id not found in the list of matched TAs are associated with students unique_id...")
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

    combined_df = combined_df.merge(
        TAs_apps_filtered[['unique_id','TA_role', 'certificate_type', 'career_status']],
        left_on='uid',right_on='unique_id',
        how='left')

    # Step 5: Check for duplicates within each WeekDay and UID combination
    duplicates = combined_df[combined_df.duplicated(subset=['WeekDay', 'uid'], keep=False)]
    if not duplicates.empty:
        print(f"Duplicate entries found based on ['WeekDay', 'uid']: {len(duplicates)}")
        combined_df = combined_df.drop_duplicates(subset=['WeekDay', 'uid'], keep='first')
        print("Removed duplicates. New shape:", combined_df.shape)
        print('-' * 80)
        print(' ')

    # Step 6: Save the cleaned DataFrame to CSV
    combined_df.to_csv(output_file, index=False)
    print(f"Cleaned DataFrame saved to {output_file}")

    return combined_df
a
```

## Call the function

```python
combined_df = clean_and_combine_data(
              dataframes_dict   = dataframes_dict,
              TAs_apps_path     = "/content/drive/MyDrive/Academies_DataAnalysis/General/TAs_ReceivedApp_from2021.csv", # to be updated in 2025
              course_id         = 13,       # Specify the course_id number of interest
              status            = "matched",   # Do NOT change the status
              ST_apps_path      = "/content/drive/MyDrive/Academies_DataAnalysis/General/Students_ReceivedApp_from2021.csv", # to be updated in 2025
              certificates_path = "/content/drive/MyDrive/Academies_DataAnalysis/General/Certificate2024.csv", # to be updated in 202
              output_file       = "cleaned_combined_df.csv")
```