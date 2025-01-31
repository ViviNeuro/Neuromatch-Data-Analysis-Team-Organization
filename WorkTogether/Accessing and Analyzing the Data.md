# Accessing and Analyzing the Data

To perform analyses, you’ll need to access the datasets stored in Google Drive. Here’s how to get started:

## 1. Locate the Data  
The data are organized in Google Drive and structured by year. Each folder corresponds to a specific year of Academies data collection.

**Link to Google Drive**:  
📂 [**Academies_DataAnalysis**](https://drive.google.com/drive/folders/1MZPn-ceDuyqoCc-gdiKsAw336Swg8qzZ)

If you do not have access, please reach out to **Viviana Greco** or **Courtney Dean**.

---

## 2. Use Google Colab to Access Google Drive  
Using Google Colab, follow these steps to mount Google Drive and access the data:

1. First, add a shortcut of the **Academies_DataAnalysis** folder to your Drive:
   - Go to the **"Shared with me"** section in Google Drive.
   - Right-click the **Academies_DataAnalysis** folder.
   - Click **"Add shortcut to Drive"** → Select **"All locations"** → Choose **"My Drive"** → Click **"Add"**.

2. Use the following code snippet in Google Colab to mount Google Drive and load the data:

```python
from google.colab import drive
import pandas as pd

# Mount Google Drive
drive.mount('/content/drive')

# Set the path to the data folder
data_path = "/content/drive/My Drive/Academies_DataAnalysis/July2024/"

# Example: Load attendance data for 2024
attendance = pd.read_csv(data_path + "attendance_2024.csv")
print(attendance.head())
```