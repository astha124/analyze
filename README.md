# Data Processing Project

This project provides an automated pipeline for converting and processing data, along with a CI/CD workflow to ensure code quality and publish results.

## Project Structure

- `data.xlsx`: The initial Excel data file. This file is converted to `data.csv` for processing.
- `data.csv`: The CSV version of the data, ready for `execute.py`.
- `execute.py`: A Python script responsible for processing `data.csv` and generating `result.json`.
- `index.html`: A single-file, responsive web page using Tailwind CSS to provide an overview of the project and a link to the processed results.
- `LICENSE`: The MIT License for this project.
- `.github/workflows/ci.yml`: GitHub Actions workflow for linting, processing, and publishing results.

## `data.csv`

```csv
ID,Category,Value,Timestamp
1,A,100,2023-01-01T10:00:00Z
2,B,150,2023-01-01T11:00:00Z
3,A,200,2023-01-01T12:00:00Z
4,C,75,2023-01-01T13:00:00Z
5,B,abc,2023-01-01T14:00:00Z
6,A,300,2023-01-01T15:00:00Z
7,C,,2023-01-01T16:00:00Z
```

## `execute.py`

This script reads `data.csv`, processes the data, and outputs a `result.json` file. A non-trivial error was identified and fixed in the original script. The fix specifically addresses robust numerical conversion and handling of missing values in the 'Value' column, ensuring that operations like `sum()` and `mean()` do not fail due to invalid data types or `NaN` values. The script now gracefully converts values to numeric types, coercing errors to `NaN`, and then fills `NaN` values with `0` to allow for successful aggregation.

```python
import pandas as pd
import json
import sys

# Ensure compatibility with Python 3.11+
if sys.version_info < (3, 11):
    sys.exit("This script requires Python 3.11 or higher.")

# Main data processing logic
try:
    # Read the data.csv file
    df = pd.read_csv('data.csv')

    # Fix for non-trivial error: Ensure 'Value' column is numeric
    # and handle potential errors or missing values.
    if 'Value' in df.columns:
        # Convert 'Value' column to numeric, coercing errors to NaN
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')
        
        # Fill NaN values with 0 (or another appropriate strategy like mean/median)
        # This ensures aggregations don't fail due to missing data.
        df['Value'] = df['Value'].fillna(0)

        # Perform aggregation
        result = {
            'total_sum_of_values': df['Value'].sum(),
            'average_value': df['Value'].mean(),
            'record_count': len(df),
            'categories': df['Category'].nunique()
        }
    else:
        result = {
            'error': 'Column "Value" not found in data.csv', 
            'record_count': len(df)
        }

    # Write the result to result.json
    with open('result.json', 'w') as f:
        json.dump(result, f, indent=2)

except FileNotFoundError:
    print("Error: data.csv not found. Please ensure the file exists.", file=sys.stderr)
    sys.exit(1)
except ImportError:
    print("Error: pandas not found. Please install it using 'pip install pandas'.", file=sys.stderr)
    sys.exit(1)
except Exception as e:
    print(f"An unexpected error occurred during execution: {e}", file=sys.stderr)
    sys.exit(1)
```

## GitHub Actions Workflow (`.github/workflows/ci.yml`)

A GitHub Actions workflow is set up to automate the following steps on every push to the `main` branch:

1.  **Checkout Code**: Retrieves the repository content.
2.  **Setup Python 3.11**: Configures the environment with Python 3.11.
3.  **Install Dependencies**: Installs `pandas` and `ruff`.
4.  **Run Ruff Linter**: Checks the Python code for style and potential errors, displaying results in the CI log.
5.  **Execute Script**: Runs `python execute.py > result.json` to generate the `result.json` output.
6.  **Upload Artifact**: Uploads the generated `result.json` as an artifact for GitHub Pages deployment.
7.  **Deploy to GitHub Pages**: Publishes the `result.json` file to GitHub Pages, making it accessible via a public URL.

### Workflow Details

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch: # Allows manual triggering

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    permissions:
      contents: write # For checkout and artifact upload
      pages: write   # For deploying to GitHub Pages
      id-token: write # For authentication with GitHub Pages

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas ruff

      - name: Run Ruff Linter
        run: ruff check .

      - name: Execute data processing script
        run: python execute.py > result.json

      - name: Upload artifact for GitHub Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.' # Uploads result.json and other files

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Running Locally

To run `execute.py` locally, ensure you have Python 3.11+ installed and then:

1.  **Install dependencies**:
    ```bash
    pip install pandas
    ```
2.  **Create `data.csv`**: Make sure the `data.csv` file (as shown above) is in the same directory as `execute.py`.
3.  **Execute the script**:
    ```bash
    python execute.py
    ```
    This will generate `result.json` in the current directory.

## View Results

The `index.html` file provides a simple web interface. After the CI/CD pipeline runs and deploys to GitHub Pages, you can access `result.json` directly from your repository's GitHub Pages URL. For example, if your repository is `username/repo`, the URL might be `https://username.github.io/repo/result.json`.

---
