
---
title: "Family History Library Catalog Metadata Clean-up"
author: "Talon Hintze, Sam Anderson, Karen Castillo, Dali Li, Z"
execute:
  warning: false
  fig.width: 5
  fig.height: 5 
format:
  html:
    theme: "darkly"
    highlight: "pygments"
    self-contained: true
    page-layout: full
    title-block-banner: true
    toc: true
    toc-depth: 3
    toc-location: body
    number-sections: false
    html-math-method: katex
    code-fold: true
    code-summary: "Show the code"
    code-overflow: wrap
    code-copy: hover
    code-tools:
        source: false
        toggle: true
        caption: See code
---

```{python}

import polars as pl
from lets_plot import *
LetsPlot.setup_html()
import pandas as pd
import re
import numpy as np
import matplotlib.pyplot as plt
from IPython.display import display, HTML
from tabulate import tabulate
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

raw = pl.read_excel("D:\\School\\Fall24\\Data Science Consulting\\xlsx data\\1-45k-A.xlsx")

data = pl.read_csv("D:\\School\\Fall24\\Data Science Consulting\\xlsx data\\combined.csv", ignore_errors=True)
data = data.with_row_index(name="index")

small = data.select(["index", "040$b-Language of cataloging", "264$a-Place of production, publication, distribution, manufacture", "264$b-Name of producer, publisher, distributor, manufacturer", "264$c-Date of production, publication, or distribution"])

def remove_non_special_chars(df: pl.DataFrame, column_names: list) -> pl.DataFrame:
    # Define the regex pattern to keep only the specified special characters
    pattern = r"[^@_!#$%^&*()<>?/\|}{~:]"
    
    # Loop through each column name in the provided list
    for column_name in column_names:
        # Apply the regex pattern to the column, replacing everything except special characters with an empty string
        df = df.with_columns(
            pl.col(column_name).str.replace_all(pattern, "").alias(column_name)
        )
    
    return df

def remove_numbers(df: pl.DataFrame, column_names:list) -> pl.DataFrame:
    # Define the regex pattern for numbers (digits 0-9)
    pattern = r"\d"
    
    # Apply the regex pattern to the column, replacing numbers with an empty string
    for column_name in column_names:

        df = df.with_columns(
            pl.col(column_name).str.replace_all(pattern, "").alias(column_name)
        )
    
    return df

```

# Introduction

The purpose of this document is to give an explanation of the methodologies used to achieve 3 main goals.

1. Identify and compare record formats
2. Compare various language columns
3. Identify 

This document walks through the steps taken to acieve these goals with the intent that they can be replicated. They may also require some review by domain experts to ensure data is being handled effectively and appropriately. 

# Data Preprocessing

Data is exported in the MARC bibliographic format. This format is used by libraries to store metadata about books, journals, and other materials. Initially, the data looks something like this:

```{python}
print(raw.head())
```

Unless someone is very familiar with MARC, these columns names don't mean anything significant. In an effor to make the data more understandable, we will rename the columns to something more meaningful. Columns were renamed according to the mark standard found [here](https://www.loc.gov/marc/bibliographic/).

These column names were simply stored in a python dictionary and then mapped to their respective columns. This gives us a better understanding of what the data means with an output that looks like this:

```{python}
print(data.head())
```

# 1. Identifying and Analyzing Formats
The Family History Library Catalog has hundreds of thousands of records. Unfortunately, these records through the years have conformed to many different formats. The goal of this project is to identify these format types and evaluate their similarity to each other.  This will help us to better understand the structure of the metadata in the Family History Library Catalog and give us more resources to employ in efforts to clean and standardize this metadata.

For a proof of concept, let's take a look at column 264 and its related subfields (a, b, c). According to the Marc21 format, these fields have to do with publications information.

* 264$a-Place of production, publication, distribution, manufacture
* 264$b-Name of producer, publisher, distributor, manufacturer
* 264$c-Date of production, publication, or distribution

These fields have a good mix of letters, digits, and special characters. As such, they make for a good proof of concept. The methods and functions used here are designed to be scaled out to different columns. Keep in mind this is a rough draft and will be refined for production use.

## Methodology
After a visual analysis of the data found in this dataset (shown below), and after collaboration, it was determined that an effective way of identifying formatting patterns would be to remove any non special characters (letters, numbers, etc). This would leave behind only the special characters that are used for formatting.

```{python}
small.head()
```
\

For example, if there is a value formatted like this: "[city, state]". It would now be represented as "[, ]". After formatting down to this format, each unique combination of special characters now represents a format type. This means that grouping by each of these unique combinations will give a count of how many records in your given sample pertain to that formatting standard.

## 264$a-Place of production, publication, distribution, manufacture

```{python}
place = remove_non_special_chars(small, ["264$a-Place of production, publication, distribution, manufacture"]).select(["index", "264$a-Place of production, publication, distribution, manufacture"])

place_agg = (
    place
    .group_by("264$a-Place of production, publication, distribution, manufacture")
    .agg(pl.len().alias("Count"))  # Apply alias to the aggregation
).sort("Count", descending=True)
```

Let's take a look at the first column of interest,  264$a. This column contains the place of publication, distribution, etc. After stripping out all but the special characters, we are left with data that looks like this.

```{python}
place.drop_nulls().head(10)
```
\

Now, let's group by this column and count the number of records for each unique combination of special characters. Please note that characters from foreign languages are not currently being treated as traditional letter characters, so they will still appear.

Here is a list of the top 10 formats.

```{python}
place_agg.sort("Count", descending=True).head(10)
```

While a list of records is certainly useful, it can be hard to digest. Let's take a look at things in a more visual format using a simple bar chart. 

```{python}
ggplot(place_agg, aes(x="264$a-Place of production, publication, distribution, manufacture", y="Count")) +\
    geom_bar(stat="identity") +\
    labs(
        title = "Distribution of Publication Formats",
        x = "Format of Place of Publication",
        y = "# of Records"
    ) +\
    theme_minimal()
```

## Comparing 264$a with 040$b-Language of cataloging

The true value of this approach is to see what kinds of formats certain types of records have. To illustrate this let's take a look at the relationship between the language of cataloging and the formats present in column 264$a.
```{python}

df2 = (
    small
    # First, remove non-special characters in specified columns
    .pipe(remove_non_special_chars, [
        "264$a-Place of production, publication, distribution, manufacture"
    ]).select(["index", "264$a-Place of production, publication, distribution, manufacture", '040$b-Language of cataloging'])
    )

test2 = (
    df2
    .group_by(
        ['264$a-Place of production, publication, distribution, manufacture', 
         '040$b-Language of cataloging']
    )
    .agg(pl.len().alias("count"))  # Ensure to name the count column
    .pivot(
        on='040$b-Language of cataloging', 
        index='264$a-Place of production, publication, distribution, manufacture', 
        values='count'
    )
)

# Convert to long format for plotting
test_long2 = test2.unpivot(
    index=['264$a-Place of production, publication, distribution, manufacture'],  # Keep this as identifier
    on=test2.columns[1:],  # Use all other columns as value variables
    variable_name='040$b-Language of cataloging',  # Name for the variable column
    value_name='count'  # Name for the value column
)

ggplot(test_long2, aes(y="040$b-Language of cataloging", x="264$a-Place of production, publication, distribution, manufacture")) +\
    geom_tile(aes(fill="count")) +\
    labs(title="264$a Formats by Language",
         x = "264$a Format",
         y = "Language Catalog") +\
    theme_minimal() +\
    theme(plot_margin=(10, 80, 10, 10)) + \
    scale_fill_gradient(low = "white", high = "blue", limits = (0, 2200))
```

This type of visualization can help us to understand which kinds of records tend to follow what kinds of formats. This knowledge will provide insight into certain types of records that will be easy and important to update. If the goal is to standardize formats across all records, a dynamic heatmap visualization like this can help to identify the best places to focus those efforts.

# 2. Title and Text Languages

To effectively utilize the bibliographic data from MARC 21, we aim to clean the dataset concerning titles and languages. This dataset includes fields such as: 
* '008 - Fixed-Length Data Elements - General Information,' 
* '040\$b - Language of Cataloging,' 
* '041\$a - Language Code of Text,' 
* '546\$a - Language Note,' 
* '245\$a - Title,' 
* '245\$b - Remainder of Title,' 
* '245\$c - Statement of Responsibility,' 
* '245\$f - Inclusive Dates,' 
* '245\$n,'
* '245\$p - Name of Part/Section of Work.' 

Currently, the information is dispersed across different columns, making it challenging to identify the correct data. The primary goal of this project is to organize the title and language columns to facilitate analysis by Family Search.

## Language columns Clean

The '008 - Fixed-Length Data Elements - General Information' field provides language information in positions 35 to 37. When multiple languages are indicated in the '008' field, only the '041\$a - Language Code of Text' field is used to represent these languages. The combined '008' and '041' fields are used when multiple languages are present in '008,' as these languages are relevant for Family Search's purposes.

The table shows that the top five languages used are English, French, German, Spanish, and Dutch.

```{python}
# %% import the file
df = data.to_pandas()
col = df.columns.tolist()

# %% drop null column
df = df.loc[:, df.notnull().any()]

# %% filter columns related to title and language
df1 = df[['008-Fixed-Length Data Elements-General Information','040$b-Language of cataloging', '041$a-Language code of text','546$a-Language note', '245$a-Title', '245$b-Remainder of title', '245$c-Statement of responsibility','245$f-Inclusive dates', '245$n', '245$p-Name of part/section of work']]
df1 = pd.DataFrame(df1)

# %% split the 008 column by position 35-37
df1['008-language'] = df1['008-Fixed-Length Data Elements-General Information'].str.slice(35, 38)

# %% 008 column - langauge result
result = df1.groupby('008-language').size().reset_index(name='count').sort_values(by='count', ascending=False)

# %% combined 008 and 041 only the case 008 have multi langauge. 
df1['008+041'] = np.where(pd.isna(df1['041$a-Language code of text']), 
                            df1['008-language'],  # Use '008-language' if '041$a-Language code of text' is NaN
                            df1['041$a-Language code of text']) 

# %% split language column by ;
df1_split2 = df1['008+041'].str.split(';', expand=True)
df1_split2.columns = ['008+041_part1', '008+041_part2', '008+041_part3','008+041_part4','008+041_part5','008+041_part6']
df1 = pd.concat([df1, df1_split2], axis=1)
df1 = df1.fillna('None')

# %% count of language code
result1 = df1.groupby('008+041').size().reset_index(name='count').sort_values(by='count', ascending=False)

result1
```

## Title columns Clean

The 245$a - Title and 245$b - Remainder of Title fields display the title and subtitle. These fields were combined and then split by the delimiter '=' to create separate columns for each value, organizing the information effectively. The langid python library was used to determine the language used in the title. This library uses a different way to detect the lanague with MARC21. It was mapped to follow MARCH 21. This is reference: <href\>https://www.loc.gov/marc/languages/language_c </href\>

The table illustrates the frequency of languages in the title column.

```{python}
# %% combined 245a and 245b (title and subtitle)
df1['245$ab'] = df1['245$a-Title'] + ' ' + df1['245$b-Remainder of title'].fillna('')

# %% split 245$ab by determilter = and make columns for each splited values
df1_split = df1['245$ab'].str.split('=', expand=True)
df1_split.columns = ['245ab_part1', '245ab_part2', '245ab_part3', '245ab_part4', '245ab_part5']
df1 = pd.concat([df1, df1_split], axis=1)

#%% check contains = in 245$ab
df1[df1['245$ab'].str.contains('=', na=False)]

#%% langauage finder library
import langid

# %%
# Function to detect the language
def detect_language(text):
    try:
        lang, _ = langid.classify(text)
        return lang
    except:
        return 'None'

# Apply the language detection to the new columns
df1['245abpart1_lan1'] = df1['245ab_part1'].apply(detect_language)
df1['245abpart2_lan2'] = df1['245ab_part2'].apply(detect_language)
df1['245abpart3_lan3'] = df1['245ab_part3'].apply(detect_language)
df1['245abpart4_lan4'] = df1['245ab_part4'].apply(detect_language)
df1['245abpart5_lan5'] = df1['245ab_part5'].apply(detect_language)

# %% count title column part : check on
value_counts1 = df1['245abpart1_lan1'].value_counts()
value_counts2 = df1['245abpart2_lan2'].value_counts()
value_counts3 = df1['245abpart3_lan3'].value_counts()
value_counts4 = df1['245abpart4_lan4'].value_counts()
value_counts5 = df1['245abpart5_lan5'].value_counts()

title_lan = pd.concat([value_counts1, value_counts2,value_counts3,value_counts4, value_counts5 ], axis=1)
title_lan = title_lan.fillna(0)
title_lan['Total'] = title_lan.sum(axis=1)
title_lan
```

```{python}
# %%
# Language code to language name mapping
language_mapping = {
    'en': 'eng',  # English
    'de': 'ger',  # German
    'es': 'spa',  # Spanish
    'fr': 'fre',  # French
    'sv': 'swe',  # Swedish
    'da': 'dan',  # Danish
    'nl': 'dut',  # Dutch
    'no': 'nor',  # Norwegian
    'pt': 'por',  # Portuguese
    'it': 'ita',  # Italian
    'fi': 'fin',  # Finnish
    'cs': 'cze',  # Czech
    'gl': 'glg',  # Galician
    'hu': 'hun',  # Hungarian
    'la': 'lat',  # Latin
    'id': 'ind',  # Indonesian
    'pl': 'pol',  # Polish
    'ms': 'may',  # Malay
    'nn': 'nno',  # Norwegian (Nynorsk)
    'sk': 'slo',  # Slovak
    'is': 'ice',  # Icelandic
    'af': 'afr',  # Afrikaans
    'cy': 'wel',  # Welsh
    'vo': 'vol',  # Volapük
    'ca': 'cat',  # Catalan
    'ro': 'rum',  # Romanian
    'lt': 'lit',  # Lithuanian
    'nb': 'nob',  # Norwegian (Bokmål)
    'eu': 'baq',  # Basque
    'sw': 'swa',  # Swahili
    'hr': 'hrv',  # Croatian
    'fo': 'fao',  # Faroese
    'et': 'est',  # Estonian
    'sl': 'slv',  # Slovenian
    'mg': 'mlg',  # Malagasy
    'lv': 'lav',  # Latvian
    'ga': 'gle',  # Irish
    'tr': 'tur',  # Turkish
    'qu': 'que',  # Quechua
    'tl': 'tgl',  # Tagalog
    'jv': 'jav',  # Javanese
    'ja': 'jpn',  # Japanese
    'lb': 'ltz',  # Luxembourgish
    'eo': 'epo',  # Esperanto
    'xh': 'xho',  # Xhosa
    'rw': 'kin',  # Kinyarwanda
    'mt': 'mlt',  # Maltese
    'an': 'arg',  # Aragonese
    'ru': 'rus',  # Russian
    'hy': 'arm',  # Armenian
    'oc': 'oci',  # Occitan (post-1500)
    'bg': 'bul',  # Bulgarian
    'se': 'sme',  # Northern Sami
    'ht': 'hat',  # Haitian French Creole
    'wa': 'wln',  # Walloon
    'zh': 'chi'   # Chinese
}

# %% apply MARC definitions to title 
df1['245abpart1_lan1'] = df1['245abpart1_lan1'].replace(language_mapping)
df1['245abpart2_lan2'] = df1['245abpart2_lan2'].replace(language_mapping)
df1['245abpart3_lan3'] = df1['245abpart3_lan3'].replace(language_mapping)
df1['245abpart4_lan4'] = df1['245abpart4_lan4'].replace(language_mapping)
df1['245abpart5_lan5'] = df1['245abpart5_lan5'].replace(language_mapping)
```

## Analysis of Language and Title

We identified four pattersn between the language and title:
1. Language and title match.
2. Cases with multiple languages in the language column but not in the title.
3. Cases with multiple languages in the title but not in the language column.
4. Cases where languages differ between the language and title columns.

```{python}
# List of the columns you're interested in
title_cols= ['245abpart1_lan1', '245abpart2_lan2', '245abpart3_lan3', '245abpart4_lan4', '245abpart5_lan5']
lan_cols = ['008+041_part1', '008+041_part2', '008+041_part3', '008+041_part4', '008+041_part5']
# %%
def compare_columns(row):
    matching_values = []
    title_not_matching = []
    lan_not_matching = []
    
    # Iterate over corresponding column pairs
    for title_col, lan_col in zip(title_cols, lan_cols):
        title_value = row.get(title_col, '')
        lan_value = row.get(lan_col, '')

        if isinstance(title_value, str) and isinstance(lan_value, str):
            if title_value == lan_value and title_value:
                matching_values.append(title_value)
            else:
                title_not_matching.append(title_value)
                lan_not_matching.append(lan_value)
    
    # Prepare the results
    matching_value_result = ', '.join(matching_values) if matching_values else 'Unmatched'
    title_not_matching_result = ', '.join(title_not_matching) if title_not_matching else 'Matched'
    lan_not_matching_result = ', '.join(lan_not_matching) if lan_not_matching else 'Matched'
    
    return matching_value_result, title_not_matching_result, lan_not_matching_result

# %%
# Apply the function across the DataFrame and expand results into new columns
df1[['matching_value', 'language_245_not_matching', 'language_008+041_not_matching']] = df1.apply(compare_columns, axis=1, result_type='expand')
# %%
# Clean the 'matching_value' column and remove 'None' entries
df1['matching_value'] = df1['matching_value'].apply(lambda x: ', '.join(value.strip() for value in x.split(',') if value.strip() != 'None'))
# %%
# Clean up 'language_245_not_matching' and 'language_008+041_not_matching'
def clean_none(value):
    return ', '.join([lang for lang in value.split(', ') if lang.strip() != 'None']) if value else 'None'

df1['language_245_not_matching'] = df1['language_245_not_matching'].apply(clean_none).fillna('None').replace('', 'None')
df1['language_008+041_not_matching'] = df1['language_008+041_not_matching'].apply(clean_none).fillna('None').replace('', 'None')

# Create a column to check if both 'language_245' and 'language_008+041' are matching
df1['both_matching'] = (df1['language_245_not_matching'] == 'Matched') & (df1['language_008+041_not_matching'] == 'Matched')
```

## Results 
We found that most cases fall under Case 1, where both the title and language records match, totaling 6,471 instances. In these cases, multiple languages are present in the record, not just in the title. Case 3, with 74 instances, has multiple languages in the title that are not reflected in the language column. Case 4, with 11,442 instances, shows different languages between the title and language column.

Cases 3 and 4 require careful consideration to understand why these discrepancies occur. In particular, case 4 may involve errors, as the Python library might not accurately detect the correct language from the title.

```{python}
# %% Find the cases
filtered = df1[['245$a-Title','matching_value', 'language_245_not_matching', 'language_008+041_not_matching']]

case1 = filtered[(filtered['language_245_not_matching'] == "Matched") & (filtered['language_008+041_not_matching'] == "Matched")]
case2 = filtered[filtered['language_245_not_matching']== "None" ] 
case3 = filtered[filtered['language_008+041_not_matching']== "None" ] 
case4 = filtered[filtered['matching_value']== "" ] 

# %% final result
# Count rows in each case
counts = {
    "Case 1": case1.shape[0],
    "Case 2": case2.shape[0],
    "Case 3": case3.shape[0],
    "Case 4": case4.shape[0]
}

# Convert to DataFrame for plotting
counts_df = pd.DataFrame(list(counts.items()), columns=['Case', 'Count'])

# Plot
plt.figure(figsize=(8, 6))
bars = plt.bar(counts_df['Case'], counts_df['Count'], color='skyblue')

# Add counts on top of each bar
for bar in bars:
    yval = bar.get_height()
    plt.text(bar.get_x() + bar.get_width()/2, yval, int(yval), ha='center', va='bottom')

plt.xlabel("Case")
plt.ylabel("Count")
plt.title("Number of Rows per Case")
plt.show()

# %% save as excel
# with pd.ExcelWriter('output_cases.xlsx') as writer:
#     df1.to_excel(writer, sheet_name='All', index=True)
#     case1.to_excel(writer, sheet_name='Case1', index=True)
#     case2.to_excel(writer, sheet_name='Case2', index=True)
#     case3.to_excel(writer, sheet_name='Case3', index=True)
#     case4.to_excel(writer, sheet_name='Case4', index=True)

```

# 3. Date Pattern Analysis
This analysis examines entries in the 245$f-Inclusive dates field of MARC records, categorizing them based on specific date patterns. The workflow consists of:

* Initial Data Inspection: Conducts an initial exploration of the dataset, including missing value identification.

* Date Pattern Categorization: Groups date entries by common formats such as ranges and single years.

* Data Cleaning: Removes special characters and analyzes the cleaned dataset for consistency.

* Special Character Analysis: Counts and assesses the presence of special characters.

* Detailed 'Other' Pattern Breakdown: Categorizes complex date formats in the "Other" group.

## Initial Data Exploration
We start by loading the data and exploring the first few rows of the 245$f-Inclusive dates column to understand its structure and identify missing values.

```{python}
# Load the data
df = data.to_pandas()

# Step 1: Initial exploration of the data
initial_data_df = df[['245$f-Inclusive dates']].head(10)
missing_values_count = df['245$f-Inclusive dates'].isnull().sum()
unique_values = df['245$f-Inclusive dates'].unique()[:10]


```

### Observations
* The dataset includes missing values and diverse date formats.
* Initial rows and missing values are summarized below:

```{python}
# Display initial exploration results
exploration_summary = pd.DataFrame({
    "Metric": ["Missing Values", "Unique Values (First 10)"],
    "Value": [missing_values_count, unique_values]
})
display(HTML("<h3>Summary of Missing and Unique Values</h3>"))
display(HTML(exploration_summary.to_html(index=False)))

```

## Categorizing Date Patterns
We classify dates into four categories:

* Missing: No date provided.
* Date Range: A range in the format YYYY-YYYY.
* Single Year: A single year in the format YYYY.
* Other: Any other format.

```{python}
# Step 2: Categorize date patterns
def categorize_date_pattern(date):
    if pd.isnull(date):
        return 'Missing'
    elif re.match(r'^\d{4}-\d{4}$', date):
        return 'Date Range'
    elif re.match(r'^\d{4}$', date):
        return 'Single Year'
    else:
        return 'Other'

df['date_pattern'] = df['245$f-Inclusive dates'].apply(categorize_date_pattern)

# Count occurrences of each pattern
date_pattern_counts = df['date_pattern'].value_counts()
total_date_patterns = date_pattern_counts.sum()
date_pattern_df = pd.DataFrame({
    'Date Pattern': date_pattern_counts.index,
    'Count': date_pattern_counts.values,
    'Percentage': (date_pattern_counts / total_date_patterns * 100).round(2)
})
```

## Distribution of Date Patterns
The table below summarizes the distribution of date patterns:

```{python}
# Display pattern distribution
display(HTML("<h3>Date Pattern Distribution</h3>"))
display(HTML(date_pattern_df.to_html(index=False)))

```

## Visualizing Date Patterns
The distribution of date patterns is shown in the bar chart below. Annotations highlight examples for each category.

```{python}
# Step 3: Visualize date patterns
plt.figure(figsize=(10, 5))
bars = plt.bar(date_pattern_counts.index, date_pattern_counts.values, color=['blue', 'green', 'orange', 'red'])
plt.xlabel('Date Pattern Category')
plt.ylabel('Count of Records')
plt.title('Distribution of Date Patterns in 245$f-Inclusive dates')
plt.xticks(rotation=45)

# Add annotations with examples
annotations = {
    'Missing': 'Examples:\nNaN\n...',
    'Date Range': 'Examples:\n"1549-1802",\n"1675-1811"',
    'Single Year': 'Examples:\n"1703",\n"1905"',
    'Other': 'Examples:\n"1585-1624, 1768-1804",\n"1624-1840 :"'
}
for i, bar in enumerate(bars):
    yval = bar.get_height()
    plt.text(bar.get_x() + bar.get_width() / 2, yval + 500, annotations[date_pattern_counts.index[i]], ha='center', va='bottom', fontsize=9)

plt.show()

```

## Special Character Analysis
We analyze the cleaned data by removing non-special characters to focus on formatting.

```{python}
# Step 4: Remove non-special characters
def remove_non_special_chars(df, column_name):
    pattern = r"[^@_!#$%^&*()<>?/\|}{~:]"
    df[column_name] = df[column_name].str.replace(pattern, "", regex=True)
    return df

df['245$f-Cleaned'] = df['245$f-Inclusive dates']
df = remove_non_special_chars(df, '245$f-Cleaned')

# Step 5: Count occurrences of special characters
def count_special_characters(column_data):
    char_counts = {}
    special_char_pattern = re.compile(r"[@_!#$%^&*()<>?/\|}{~:]+")
    for entry in column_data.dropna():
        matches = special_char_pattern.findall(entry)
        for match in matches:
            char_counts[match] = char_counts.get(match, 0) + 1
    char_df = pd.DataFrame(list(char_counts.items()), columns=['Character Sequence', 'Count']).sort_values(by='Count', ascending=False)
    char_df['Percentage'] = (char_df['Count'] / char_df['Count'].sum() * 100).round(2)
    return char_df

char_df = count_special_characters(df['245$f-Cleaned'])

```

### Results
Below is a table of the most frequent special characters and their percentages:

```{python}
# Display special character analysis
display(HTML("<h3>Special Character Analysis</h3>"))
display(HTML(char_df.to_html(index=False)))

```

## Detailed Analysis of 'Other' Patterns
Within the "Other" category, we perform an additional breakdown to identify sub-patterns.
```{python}
# Step 6: Analyze 'Other' patterns
def categorize_other_pattern(date):
    if re.match(r'^\d{4}-\d{4},\s?\d{4}-\d{4}$', date):
        return 'Date Range with Commas'
    elif re.match(r'^\d{1,2}/\d{1,2}/\d{4}\s\d{1,2}:\d{2}', date):
        return 'Datetime Format'
    elif re.match(r'^\d{4}-\d{4}\s?[.:]$', date):
        return 'Date Range with End Punctuation'
    elif re.match(r'^\d{4},\s?\d{4}$', date):
        return 'Single Years with Comma'
    else:
        return 'Other Unspecified Pattern'

df_other = df[df['date_pattern'] == 'Other'].copy()
df_other['Other_pattern'] = df_other['245$f-Inclusive dates'].apply(categorize_other_pattern)

# Summarize 'Other' patterns
other_pattern_counts = df_other['Other_pattern'].value_counts()
detailed_other_df = pd.DataFrame({
    'Other Pattern': other_pattern_counts.index,
    'Count': other_pattern_counts.values
})
```

## 'Other' Pattern Summary
```{python}
# Display detailed 'Other' pattern breakdown
display(HTML("<h3>Detailed 'Other' Pattern Breakdown</h3>"))
display(HTML(detailed_other_df.to_html(index=False)))

```

## Conclusion
This analysis provides insights into date formatting within MARC records. It identifies common patterns and highlights opportunities for data standardization. The cleaned and categorized data can be exported for further use.

## Date Pattern Analysis Based on Record Type
This section categorizes date patterns and creates a contingency table to analyze the relationship between publication status and date patterns.

### Step 1: Categorize Date Patterns
Date patterns in the 245$f-Inclusive dates field are categorized into four groups: Missing, Date Range, Single Year, and Other.

```{python}
# Redefine df_combined as df (if no modifications are needed from preprocessing)
df_combined = df

# Redefine df_combined as df
df_combined = df

# Step 1: Categorize Date Patterns
def categorize_date_pattern(date):
    if pd.isnull(date):
        return 'Missing'
    elif re.match(r'^\d{4}-\d{4}$', date):
        return 'Date Range'
    elif re.match(r'^\d{4}$', date):
        return 'Single Year'
    else:
        return 'Other'

# Apply the function to the '245$f-Inclusive dates' column in df_combined
df_combined['date_pattern'] = df_combined['245$f-Inclusive dates'].apply(categorize_date_pattern)

# Step 2: Recreate 'Publication Status' if needed
if '008-Fixed-Length Data Elements-General Information' in df_combined.columns:
    df_combined['Publication Status'] = df_combined['008-Fixed-Length Data Elements-General Information'].str[6]
else:
    print("'Publication Status' column could not be generated because '008-Fixed-Length Data Elements-General Information' is missing.")
```

### Step 2: Create Contingency Table
A contingency table shows the distribution of date patterns across different publication statuses.

```{python}
# Step 3: Create a contingency table
if 'Publication Status' in df_combined.columns and 'date_pattern' in df_combined.columns:
    pattern_counts = df_combined.groupby(['Publication Status', 'date_pattern']).size().unstack(fill_value=0)
    print("Exact Counts of Date Patterns for Each Record Type:")
    print(tabulate(pattern_counts, headers='keys', tablefmt='psql'))
else:
    print("Either 'Publication Status' or 'date_pattern' column is missing. Contingency table cannot be generated.")
```

# 4. Leader Bilbliography and Child/Parent Analysis

I Identify incorrect entries in: 
- 008-Leader column (7th place in cell which indicates bibliography)

- 773$w column (code of the parent record)

- 336-338 columns with Record type info

## Introduction
The book records catelog at family search have records indexed throughout the years with different standards of indexing, and may also contain mistakes. My goal is to analyize records that do not meet the MARC standards, or have invalid values, or contridictory entries.

## 000-Leader column (7th places which indicates bibliography)

The column each place in the "000-Leader" represents different information. The 7th place contains the Bibliographic level. Valid letters are 'a', 'm', 's', 'b'.

A - Monographic component part

b - Serial component part

c - collection

d - Subunit

i - Intergrating resource

m - Monograph/item

s - Serial

Invalid entries are - t, - k, - e,- i,- g,- d. If we look at the rows with invalid values in the 7th columnm, and then look at the 8th column, we see valid entries that should be in the 7th place.
Most likely there was an issue during exporting the data that caused some of the entries to be off by one place value.


```{python}
import pandas as pd
import numpy as np
from tabulate import tabulate

# Step 1: Define file path and read CSV with error handling
csv_file_path = 'D:\\School\\Fall24\\Data Science Consulting\\xlsx data\\combined.csv'
try:
    df = pd.read_csv(csv_file_path, on_bad_lines='skip', low_memory=False)  # Handles bad lines gracefully
except pd.errors.ParserError as e:
    print(f"Error reading CSV: {e}")
    raise


# Step 2: Clean data by removing columns with only NA values
df_cleaned = df.dropna(axis=1, how='all')

# Step 3: Filter columns with specific prefixes
prefixes = [
    '000','001', '008', '245$6', '245$a', '245$b', '245$f', '245$n', '245$p',
    '260$a', '260$b', '260$c',
    '264$a', '264$b', '264$c', '773$w'
]
df_filtered = df_cleaned[[col for col in df_cleaned.columns if any(col.startswith(prefix) for prefix in prefixes)]]

# Step 4: Split '008-Fixed-Length Data Elements-General Information' into separate columns
if '008-Fixed-Length Data Elements-General Information' in df_filtered.columns:
    df_split = df_filtered['008-Fixed-Length Data Elements-General Information'].apply(
        lambda x: pd.Series({
            'Record Creation Date': x[:6] if len(x) >= 6 else np.nan,
            'Publication Status': x[6] if len(x) >= 7 else np.nan,
            'Date 1': x[7:11] if len(x) >= 11 else np.nan,
            'Date 2': x[11:15] if len(x) >= 15 else np.nan,
            'Place of Publication': x[15:18] if len(x) >= 18 else np.nan,
            'Language': x[35:38] if len(x) >= 38 else np.nan,
            'Modified Record': x[38] if len(x) >= 39 else np.nan
        })
    )
    df_combined = pd.concat([df_filtered, df_split], axis=1)
else:
    df_combined = df_filtered

# Step 5: Add 'Bibliography' column from '000-Leader'
df_combined['Bibliography'] = df_combined['000-Leader'].str[6]  # 7th character is at index 6
df_combined['6th'] = df_combined['000-Leader'].str[5]  # 6th character (index starts from 0)
df_combined['8th'] = df_combined['000-Leader'].str[7]  # 8th character

# Step 6: Filter rows based on 'Bibliography' values
exclude_chars = ['a', 'm', 's', 'b']
filtered_df = df_combined[~df_combined['Bibliography'].isin(exclude_chars)]

# Step 7: Select specific columns and add new columns from '000-Leader'
result_df = filtered_df[['000-Leader', 'Bibliography', '6th', '8th']].copy()

# Step 8: Count values for specific columns (excluding the '6th' column)
def print_counts(column_name):
    value_counts = result_df[column_name].value_counts()
    print(f"\nCount of each distinct value in the {column_name} column:")
    print(tabulate(value_counts.reset_index().values, headers=['Value', 'Count'], tablefmt='pretty'))

for col in ['Bibliography', '8th']:
    print_counts(col)

# Step 9: Check if total count of 'Bibliography' matches total count of '8th' column
bibliography_total_count = result_df['Bibliography'].count()
eighth_total_count = result_df['8th'].count()
print(f"\nTotal count in Bibliography column: {bibliography_total_count}")
print(f"Total count in 8th character column: {eighth_total_count}")
if bibliography_total_count == eighth_total_count:
    print("The counts match.")
else:
    print("The counts do not match.")

```

```{python}
# Step 10: Export filtered rows to a CSV file with '6th' and '8th' columns from result_df, including '001-Control Number'
output_csv_path = '/content/filtered_bibliography.csv'
filtered_export_df = filtered_df.merge(result_df[['000-Leader', '6th', '8th']], on='000-Leader', how='left')

desired_columns = ['000-Leader', 'Bibliography', '6th', '8th', '001-Control Number']
available_columns = [col for col in desired_columns if col in filtered_export_df.columns]
filtered_export_df = filtered_export_df[available_columns]

# Export to CSV
#filtered_export_df.to_csv(output_csv_path, index=False)
#print(f"\nFiltered rows exported to {output_csv_path}")

```

## 008-Fixed-Length Data Elements-General Information
This column contains all of the following in one column:
 -Record Creation Date, 
 -Publication Status,
 -Date 1,
 -Date 2,
 -Place of Publication,
 -Language,
 -Modified Record
 *There is space for other information for records pertaining to maps, music, and a few other things but those things are not in the dataset

 I split the column into 7 column, by the fields noted above.

```{python}

# Step 11: Stats on '008-Fixed-Length Data Elements'
# Step 11.1: Filter to only include specific columns
df_combined_filtered = df_combined[[
    '008-Fixed-Length Data Elements-General Information',
    'Record Creation Date',
    'Publication Status',
    'Date 1',
    'Date 2',
    'Place of Publication',
    'Language',
    'Modified Record'
]].copy()

# Step 11.2: Create a new DataFrame that excludes specific values in 'Publication Status'
exclude_values = ['b', 'c', 'd', 'i', 'k', 'm', 'n']
df_combined_filtered = df_combined_filtered[~df_combined_filtered['Publication Status'].isin(exclude_values)].copy()

# Step 11.3: Show count of distinct values for 'Publication Status' and 'Language'
value_counts_publication_status = df_combined_filtered['Publication Status'].value_counts()
print("\nCount of each distinct value in the 'Publication Status' column:")
print(tabulate(value_counts_publication_status.reset_index().values, headers=['Publication Status', 'Count'], tablefmt='pretty'))

value_counts_language = df_combined_filtered['Language'].value_counts()
#print("\nCount of each distinct value in the 'Language' column:")
#print(tabulate(value_counts_language.reset_index().values, headers=['Language', 'Count'], tablefmt='pretty'))

```

Rows with invalid entries for the bibliographic info (7th character in the column) all have a valid entry but happens to be in the 8th space. The same amount of invalid entries in the 7th space equals the same amount of entries (which are valid) in the 8th space.

## 773$w column (code of the parent record)

773$w column contains the parent record code. We can match the 773$w with the 0001-Leader column to find the parent record row. The 773$w column is empty for many columns, but let's look at how many parent-child records match up


```{python}
# Step 12: Analysis on '773$w' and related columns
# Step 12.1: Filter rows where '773$w' is not null
filtered_df_773w = df_combined[df_combined['773$w'].notna()]

# Step 12.2: Select only the desired columns
filtered_columns_df = filtered_df_773w[['000-Leader', '001-Control Number', '773$w', 'Bibliography', '8th']]

#filtered_columns_df
#28 rows have data in the 773$w column
```


```{python}
# Step 12.3: Drop NA values from '773$w' to focus on actual values
values_in_773w = df_combined['773$w'].dropna()

# Step 12.4: Filter rows where '001-Control Number' matches a value in '773$w'
matching_rows = df_combined[df_combined['001-Control Number'].isin(values_in_773w)]

# Step 12.5: Create a new column 'Parent Control Number' in df_combined to store the matched '001-Control Number' values
df_combined['Parent Control Number'] = df_combined['773$w'].apply(lambda x: x if x in matching_rows['001-Control Number'].values else np.nan)

# Step 12.6: Filter and display the columns of interest where 'Parent Control Number' is not NaN
matched_df = df_combined[df_combined['Parent Control Number'].notna()][['000-Leader', '001-Control Number', '773$w', 'Parent Control Number', '245$a-Title']]
print("\nChild Records with existing Parent Records:")
matched_df

```


```{python}
# Step 12.7: Filter rows where '773$w' is not in '001-Control Number'
unmatched_df = df_combined[df_combined['773$w'].isin(values_in_773w) & ~df_combined['773$w'].isin(df_combined['001-Control Number'])]

# Step 12.8: Select and display the desired columns
unmatched_df_filtered = unmatched_df[['000-Leader', '001-Control Number', '773$w', 'Parent Control Number','245$a-Title']]
print("\nChild Records without exisiting Parent Records in data :")
unmatched_df_filtered

```

Aboe are the records that we are missing the Parent records. Those records will need to be found, as they are not in our dataset.

```{python}
# Step 12.9: Export unmatched rows to a CSV file
#unmatched_output_csv_path = '/content/unmatched_rows.csv'
#unmatched_df_filtered.to_csv(unmatched_output_csv_path, index=False)
#print(f"\nUnmatched rows exported to {unmatched_output_csv_path}")

```

## 337 through 338, columns with Record type info

These columns contain information on the record type of the books, which include if there is text, if it is a phsyical or eletronic record, and how many pages/volumes/leaves. I will look at if all the columns for any given row suggests there are the same amount of records.

One record can have multiple record types, and/or show that there are multiple volumes of a work.

```{python}
# Step 13: Analysis on '336$2' and related columns
# Step 13.1: Filter rows where '336$2' is not null
filtered_df_336 = df_cleaned[df_cleaned['336$2'].notna()]

# Step 13.2: Select only the desired columns
filtered_columns_df_336 = filtered_df_336[['000-Leader','300$a-Extent', '300$b-Other physical details', '300$c-Dimensions', '310$a',
                                           '336$2', '336$a', '336$b', '337$2', '337$a', '337$b',
                                           '338$2', '338$a', '338$b', '362$a-Start date of publication']]
print("\nFiltered DataFrame with non-null '336$2':")
filtered_columns_df_336

```


```{python}
# Step 13.3: Display distinct values in specified columns
distinct_values = {}
columns_to_check = ['336$2', '336$a', '336$b', '337$2', '337$a', '337$b', '338$2', '338$a', '338$b', '362$a-Start date of publication']

for col in columns_to_check:
    if col in filtered_columns_df_336.columns:
        distinct_values[col] = filtered_columns_df_336[col].dropna().unique()

distinct_values_df = pd.DataFrame.from_dict(distinct_values, orient='index').transpose()
print("\nDistinct values in specified columns:")
distinct_values_df

```

The following code shows all variations in entries for 336-338 columns. It’s commented out.
```{python}
# Step 13.4: Display distinct values for each of the specified columns individually
# distinct_values_dict = {}
# columns_to_check = ['336$b', '337$2', '337$a', '337$b', '338$2', '338$a', '338$b']

# for col in columns_to_check:
#     if col in filtered_columns_df_336.columns:
#         distinct_values_dict[col] = filtered_columns_df_336[col].dropna().unique()

# for col, values in distinct_values_dict.items():
#     print(f"Distinct Values in Column '{col}':")
#     for value in values:
#         print(value)
#     print("\n")

```


Here, I created new columns showing the count of how many records there are according to each previous column because some columns have a contradictory amount of records.

```{python}
# Step 13.5: Create new columns to count the occurrences of ';' in each of the specified columns
columns_to_analyze = ['001-Control Number', '337$a', '337$b', '338$2', '338$a', '338$b']

for col in columns_to_analyze:
    if col in filtered_columns_df_336.columns:
        count_col_name = f"{col}_count"
        filtered_columns_df_336[count_col_name] = filtered_columns_df_336[col].apply(lambda x: str(x).count(';') if pd.notna(x) else 0)

#print("\nFiltered DataFrame with count columns:")
#filtered_columns_df_336

```



```{python}
# Step 14: Create a DataFrame with specified columns
final_columns = [
    '000-Leader', '336$2', '336$a', '336$b', '337$2', '337$a', '337$b',
    '338$2', '338$a', '338$b', '362$a-Start date of publication',
    '337$a_count', '337$b_count', '338$2_count', '338$a_count', '338$b_count'
]

# Add '001-Control Number' only if it is in the columns of filtered_columns_df_336
if '001-Control Number' in filtered_columns_df_336.columns:
    final_columns.insert(1, '001-Control Number')

# Use only the columns that are available in filtered_columns_df_336
available_final_columns = [col for col in final_columns if col in filtered_columns_df_336.columns]
final_df = filtered_columns_df_336[available_final_columns].copy()

print("\nFinal DataFrame with specified columns:")
final_df

```

Rows filtered so we can see the records that have a contradictary amount of record types. For example, a column may say there are 3 record types, but another column may say there are 4 record types.


```{python}
# Step 13.5: Create new columns to count the occurrences of ';' in each of the specified columns
columns_to_analyze = ['001-Control Number', '337$a', '337$b', '338$2', '338$a', '338$b']

# Initialize count columns in filtered_columns_df_336 for each analyzed column
for col in columns_to_analyze:
    if col in filtered_columns_df_336.columns:
        count_col_name = f"{col}_count"
        filtered_columns_df_336[count_col_name] = filtered_columns_df_336[col].apply(lambda x: str(x).count(';') if pd.notna(x) else 0)

#print("\nFiltered DataFrame with count columns:")
#filtered_columns_df_336

# Add 1 to each cell in the specified count columns
count_columns = ['337$a_count', '337$b_count', '338$2_count', '338$a_count', '338$b_count']

# Increment the values in each of the specified columns by 1
for col in count_columns:
    if col in filtered_columns_df_336.columns:
        filtered_columns_df_336[col] += 1  # Add 1 to each cell in the count columns

# Ensure that '001-Control Number' is included in filtered_columns_df if it exists in df_cleaned
if '001-Control Number' in df_cleaned.columns:
    filtered_columns_df_336['001-Control Number'] = df_cleaned['001-Control Number']

# Define columns to display, including '001-Control Number' and incremented count columns
columns_to_display = ['001-Control Number'] + count_columns

# Create a condition to filter rows where values are not equal across the count columns (excluding '001-Control Number')
unequal_condition = filtered_columns_df_336[count_columns].nunique(axis=1) > 1

# Filter rows based on the condition and include '001-Control Number' in the result
unequal_rows_df = filtered_columns_df_336[unequal_condition]

# Select only the columns that are available in the DataFrame
available_columns = [col for col in columns_to_display if col in unequal_rows_df.columns]
unequal_rows_df_filtered = unequal_rows_df[available_columns]

# Display the resulting DataFrame with unequal count values, including '001-Control Number'
unequal_rows_df_filtered

```

```{python}
# Step 1: Extract the unique '001-Control Number' values from unequal_rows_df_filtered
control_numbers = unequal_rows_df_filtered['001-Control Number'].unique()

# Step 2: Define the columns you want to include in the final table
columns_to_retrieve = [
    '001-Control Number', '336$2', '336$a', '336$b', '337$2',  '337$a', '337$b', '338$2', '338$a', '338$b',
    ]

# Step 3: Filter the original DataFrame to retrieve the specified columns
# We assume that `filtered_columns_df_336` contains the necessary columns.
# If not, replace `filtered_columns_df_336` with the appropriate DataFrame name.

# Retrieve rows based on the '001-Control Number' values
final_table = filtered_columns_df_336[
    filtered_columns_df_336['001-Control Number'].isin(control_numbers)
][columns_to_retrieve]

# Display the resulting table
#final_table

```


```{python}
# Save final_table to a CSV file
#final_table.to_csv('problems_resouce_type.csv', index=False)

#print("final_table has been saved as 'problems_resouce_type.csv'")
```


```{python}
# Check if '336$2' exists in df and count non-null values if it does
if '336$2' in df.columns:
    non_null_count = df['336$2'].notna().sum()
    print(f"The number of rows with incorrect entries for cound of record types are: {non_null_count}")
else:
    print("The column '336$2' does not exist in df.")

```

# Next Steps
This is all a great start and a solid proof of concept for one potential method for identifying and analyzing different formatting standards that are used in the Family History Library Catalog metadata. This is not a comprehensive analysis, and there is much more to be done. Here are some suggestions for next steps:

The next step is to create an application in streamlit so that new data cand be processed automatically and on-demand. This will allow end users to upload newly exported data from KOHA and get specificed results without having to run the code manually. This will also allow for more flexibility and customization of the analysis.
