# Standardization Guidelines

Please follow the below standardized approaches to handle various categories in our datasets. 
The goal is to ensure data consistency, clarity, and ease of analysis by applying uniform transformations to specific fields. 
Each section describes the standard transformations to be applied to a particular category.


## Gender
To simplify and unify gender categorization, we use the following transformations:

- Replace `cis_gender` labels: Entries with `cis_gender` in combination with `male` or `female` are simplified to the respective base gender (i.e., "male" or "female").

```python
Demographics['gender'] = Demographics['gender'].replace({
    '["male", "cis_gender"]': '["male"]', 
    '["female", "cis_gender"]': '["female"]'
})
```

- Group non-binary identities: Any entry that includes `non_binary` along with another label (e.g., "female" or "male") is categorized under `non_binary.`
```python
Demographics['gender'] = Demographics['gender'].replace({
    '["female", "non_binary"]': '["non_binary"]',
    '["male", "non_binary"]': '["non_binary"]'
})
```

- Consolidate `Prefer Not to Say` responses: Entries where individuals chose `prefer_not` along with a gender label (e.g., "male" or "female") are recategorized as `prefer_not` to respect privacy.
```python
Demographics['gender'] = Demographics['gender'].replace({
    '["male", "prefer_not"]': '["prefer_not"]',
    '["female", "prefer_not"]': '["prefer_not"]'
})
```

## Dropout and No-Show Definitions

In our analysis framework:
- **Dropout**: This term refers to students who were matched to a course and attended at least one day but did not complete the requirements to receive a certificate.

- **No-Show**: This category includes matched students who did not attend any days of the course (i.e., attended zero days). No-shows are excluded from the dropout category, as they did not engage with the course at any point.

