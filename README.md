# readme


# Unstructured Analytics Final Project – \[Changing Global Catholic Populations, 1910-2010\]

## Overview

This repository contains the written portion of my final project for
Unstructured Analytics.  
It includes the Quarto document, data, and supporting code. Project
description: Looking at the differences between global Catholic
populations from 1910 to 2010, I want to understand what may have caused
or contributed to these changes in the rising and falling of numbers.

## Files

- `final_project.qmd` – Main Quarto document with code, analysis, and
  discussion.
- `data/` – Folder with the datasets used in the project.
- `README.md` – Project description and instructions.

## How to Render the Document

1.  Open the project folder in VS Code.
2.  Open `final_project.qmd`.
3.  Type quarto render in the terminal.

## Methods

-Scraping Images using beautiful soup -Optical Character Recognition
(OCR) using tesseract -Population Calculations (Growth Factor and
Percent Change) -Topic Modeling (3 types) -Sentiment Analysis (2 types)
-Regex Content Extraction -Conclusion

# Code and Comments

## Image Scraping

Here I scraped Catholic Population images from pew research that
contained statistics from 1910 and 2010(it was a very tough process of
trial and error.)

``` python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin
import os

chart_pages = [
    "https://www.pewresearch.org/religion/2013/02/13/the-global-catholic-population/pf_13-02-13_global-catholics_chart-3/",
    "https://www.pewresearch.org/religion/2013/02/13/the-global-catholic-population/pf_13-02-13_global-catholics_chart-4/"
]

headers = {"User-Agent": "Mozilla/5.0"}
os.makedirs("charts", exist_ok=True)

chart_imgs = []

for page in chart_pages:
    r = requests.get(page, headers=headers)
    soup = BeautifulSoup(r.text, "html.parser")

    img = soup.find("img")
    if not img:
        print("[!] No image found on:", page)
        continue

    src = img.get("src")
    if src:
        full = urljoin(page, src)
        chart_imgs.append(full)

print("[+] Found chart images:")
for c in chart_imgs:
    print("   ", c)

for i, img_url in enumerate(chart_imgs, start=1):

    clean_url = img_url.split("?")[0]

    img_data = requests.get(clean_url, headers=headers).content
    ext = os.path.splitext(clean_url)[1]
    filename = f"charts/chart_{i}{ext}"

    with open(filename, "wb") as f:
        f.write(img_data)

    print("[✓] Saved:", filename)
```

    [+] Found chart images:
        https://www.pewresearch.org/wp-content/uploads/sites/20/2013/02/PF_13.02.13_Global-Catholics_chart-3-1.png?w=560
        https://www.pewresearch.org/wp-content/uploads/sites/20/2013/02/PF_13.02.13_Global-Catholics_chart-4-1.png?w=560
    [✓] Saved: charts/chart_1.png
    [✓] Saved: charts/chart_2.png

Here I displayed the images I extracted to make sure that the scraping
worked.

``` python
from IPython.display import Image, display

display(Image(filename="charts/chart_1.png"))
display(Image(filename="charts/chart_2.png"))
```

![](README_files/figure-commonmark/cell-3-output-1.png)

![](README_files/figure-commonmark/cell-3-output-2.png)

# Optical Character Recognition (OCR)

I downloaded Tesseract to do OCR and scrape the statistical information
from the images

``` python
import pytesseract
from PIL import Image
import re
import pandas as pd

pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"
print(pytesseract.get_tesseract_version())
```

    5.5.0.20241111

Here I printed the scraped data from one of the images to confirm it
worked before the next step.

``` python
img = Image.open("charts/chart_2.png") 
text = pytesseract.image_to_string(img, config="--psm 6")
print(text)
```

    10 Countries with the Largest Number of Catholics, 1910

    France 40,510,000 08.4% 13.0%
    italy .35,270,000 90.0 at
    Brazil 21,430,000 056 14
    Spain 20,360,000 90.0 10
    Poland 18,750,000 aay 64
    Germany 16,580,000 387 87
    Mexico 114,280,000 o10 49
    United States 12,470,000 142 43
    Philippines 7,260,000 78.1 25
    Czech Republic 7,420,000 862 24
    World Total 291,290,000 166 100.0
    Fue for 1910 ate from the Word Christian Databaes (Er2013). Population estimate ate oundedtothetan thousands

    percentages ar calculated fom unvunded numbers. Fgwes ay rot edd excl duet rounding

    Pew Research Onter

Here I ran OCR again, created an empty table, looped through the
extracted text of data lines, checked if a line contained a population
number, extracted the info from the line, appended the extracted row,
and then returned the dataframe.

``` python
def extract_table_from_chart(path, year):
    img = Image.open(path)

    raw = pytesseract.image_to_string(img, config="--psm 6")

    rows = []

    for line in raw.splitlines():
        line = line.strip()
        if not line:
            continue

        skip_fragments = [
            "10 Countries", "Countries", "ESTIMATED", "PERCENTAGE",
            "World Total", "Population estimates", "Figures for 1910",
            "Pew Research"
        ]
        if any(s.lower() in line.lower() for s in skip_fragments):
            continue

        if not re.search(r"\d[\d,]*,\d{3},\d{3}", line) and not re.search(r"\.\d[\d,]*,\d{3},\d{3}", line):
            continue

        m_country = re.match(r"^[A-Za-z .]+", line)
        if not m_country:
            continue
        country = m_country.group(0).strip().title()

        m_pop = re.search(r"\.?\d[\d,]*,\d{3},\d{3}", line)
        if not m_pop:
            continue
        pop_str = m_pop.group(0).replace(".", "") 
        pop = int(pop_str.replace(",", ""))

        rest = line[m_pop.end():]

        nums = re.findall(r"[\d\.]+", rest)

        pct_cat = None
        pct_world = None
        if len(nums) >= 2:
            pct_cat = float(nums[-2])
            pct_world = float(nums[-1])

        rows.append({
            "Country": country,
            "Year": year,
            "Catholic_Pop": pop,
            "Percent_Catholic": pct_cat,
            "Percent_World_Catholics": pct_world
        })

    return pd.DataFrame(rows)
```

Here I printed the charts, but they got mixed up.

``` python
df_1910 = extract_table_from_chart("charts/chart_1.png", 1910)
df_2010 = extract_table_from_chart("charts/chart_2.png", 2010)

print("1910:")
display(df_1910)

print("2010:")
display(df_2010)
```

    1910:

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|  | Country | Year | Catholic_Pop | Percent_Catholic | Percent_World_Catholics |
|----|----|----|----|----|----|
| 0 | Brazil | 1910 | 126750000 | 65.0 | 11.7 |
| 1 | Mexico | 1910 | 96450000 | 85.0 | 80.0 |
| 2 | Philippines | 1910 | 78570000 | 10.0 | 70.0 |
| 3 | United States | 1910 | 75380000 | 243.0 | 10.0 |
| 4 | Aly | 1910 | 49170000 | 2.0 | 46.0 |
| 5 | Colombia | 1910 | 38100000 | 23.0 | 35.0 |
| 6 | France | 1910 | 37930000 | 60.4 | 35.0 |
| 7 | Poland | 1910 | 3310000 | 92.2 | 33.0 |
| 8 | Spain | 1910 | 34670000 | 75.2 | 32.0 |
| 9 | Democratic Republic Of The Congo | 1910 | 34210000 | 473.0 | 20.0 |

</div>

    2010:

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country        | Year | Catholic_Pop | Percent_Catholic | Percent_World_Catholics |
|-----|----------------|------|--------------|------------------|-------------------------|
| 0   | France         | 2010 | 40510000     | 8.4              | 13.0                    |
| 1   | Italy .        | 2010 | 35270000     | NaN              | NaN                     |
| 2   | Brazil         | 2010 | 21430000     | 56.0             | 14.0                    |
| 3   | Spain          | 2010 | 20360000     | 90.0             | 10.0                    |
| 4   | Poland         | 2010 | 18750000     | NaN              | NaN                     |
| 5   | Germany        | 2010 | 16580000     | 387.0            | 87.0                    |
| 6   | Mexico         | 2010 | 114280000    | 10.0             | 49.0                    |
| 7   | United States  | 2010 | 12470000     | 142.0            | 43.0                    |
| 8   | Philippines    | 2010 | 7260000      | 78.1             | 25.0                    |
| 9   | Czech Republic | 2010 | 7420000      | 862.0            | 24.0                    |

</div>

Here I corrected the information and reinforced the 1910 dataframe.

``` python
import pandas as pd

data_1910 = {
    "Country": [
        "France", "Italy", "Brazil", "Spain", "Poland",
        "Germany", "Mexico", "United States",
        "Philippines", "Czech Republic"
    ],
    "Catholic_Pop_1910": [
        40510000, 35270000, 21430000, 20350000, 18750000,
        16580000, 14280000, 12470000, 7260000, 7120000
    ],
    "Percent_Catholic_1910": [
        98.4, 99.9, 95.6, 99.9, 77.1,
        35.7, 91.0, 14.2, 78.7, 86.
    ],
    "Percent_World_1910": [
        13.9, 12.1, 7.4, 7.0, 6.4,
        5.7, 4.9, 4.3, 2.5, 2.4
    ]
}

df1910 = pd.DataFrame(data_1910)
df1910
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|  | Country | Catholic_Pop_1910 | Percent_Catholic_1910 | Percent_World_1910 |
|----|----|----|----|----|
| 0 | France | 40510000 | 98.4 | 13.9 |
| 1 | Italy | 35270000 | 99.9 | 12.1 |
| 2 | Brazil | 21430000 | 95.6 | 7.4 |
| 3 | Spain | 20350000 | 99.9 | 7.0 |
| 4 | Poland | 18750000 | 77.1 | 6.4 |
| 5 | Germany | 16580000 | 35.7 | 5.7 |
| 6 | Mexico | 14280000 | 91.0 | 4.9 |
| 7 | United States | 12470000 | 14.2 | 4.3 |
| 8 | Philippines | 7260000 | 78.7 | 2.5 |
| 9 | Czech Republic | 7120000 | 86.0 | 2.4 |

</div>

I printed the extracted information.

``` python
df_1910 = extract_table_from_chart("charts/chart_1.png", 1910)
df_2010 = extract_table_from_chart("charts/chart_2.png", 2010)
print(df_1910)
print(df_2010)
```

                                Country  Year  Catholic_Pop  Percent_Catholic  \
    0                            Brazil  1910     126750000              65.0   
    1                            Mexico  1910      96450000              85.0   
    2                       Philippines  1910      78570000              10.0   
    3                     United States  1910      75380000             243.0   
    4                               Aly  1910      49170000               2.0   
    5                          Colombia  1910      38100000              23.0   
    6                            France  1910      37930000              60.4   
    7                            Poland  1910       3310000              92.2   
    8                             Spain  1910      34670000              75.2   
    9  Democratic Republic Of The Congo  1910      34210000             473.0   

       Percent_World_Catholics  
    0                     11.7  
    1                     80.0  
    2                     70.0  
    3                     10.0  
    4                     46.0  
    5                     35.0  
    6                     35.0  
    7                     33.0  
    8                     32.0  
    9                     20.0  
              Country  Year  Catholic_Pop  Percent_Catholic  \
    0          France  2010      40510000               8.4   
    1         Italy .  2010      35270000               NaN   
    2          Brazil  2010      21430000              56.0   
    3           Spain  2010      20360000              90.0   
    4          Poland  2010      18750000               NaN   
    5         Germany  2010      16580000             387.0   
    6          Mexico  2010     114280000              10.0   
    7   United States  2010      12470000             142.0   
    8     Philippines  2010       7260000              78.1   
    9  Czech Republic  2010       7420000             862.0   

       Percent_World_Catholics  
    0                     13.0  
    1                      NaN  
    2                     14.0  
    3                     10.0  
    4                      NaN  
    5                     87.0  
    6                     49.0  
    7                     43.0  
    8                     25.0  
    9                     24.0  

Here I swapped the dataframes to make sure the data had the correct
years.

``` python
df_1910, df_2010 = df_2010.copy(), df_1910.copy()
df_1910["Year"] = 1910
df_2010["Year"] = 2010
```

Next I fixed the typos in the names of the countries

``` python
df_1910["Country"] = df_1910["Country"].replace({
    "Aly": "Italy",
    "Democratic Republic Of The Congo": "Democratic Republic of the Congo",
    "Italy .": "Italy"
})

df_2010["Country"] = df_2010["Country"].replace({
    "Aly": "Italy",
    "Italy .": "Italy"
})
```

## Truth Dictionary

Lastly I made a “truth dictionary” for each chart to make sure the data
can be easily analyzed.

``` python
truth_1910 = {
    "France": (40510000, 98.4, 13.9),
    "Italy": (35270000, 99.9, 12.1),
    "Brazil": (21430000, 95.6, 7.4),
    "Spain": (20350000, 99.9, 7.0),
    "Poland": (18750000, 77.1, 6.4),
    "Germany": (16580000, 35.7, 5.7),
    "Mexico": (14280000, 91.0, 4.9),
    "United States": (12470000, 14.2, 4.3),
    "Philippines": (7260000, 78.7, 2.5),
    "Czech Republic": (7120000, 86.2, 2.4),
}
```

``` python
truth_2010 = {
    "Brazil": (126750000, 65.0, 11.7),
    "Mexico": (96450000, 85.0, 8.9),
    "Philippines": (75570000, 81.0, 7.0),
    "United States": (75380000, 24.3, 7.0),
    "Italy": (49170000, 81.2, 4.6),
    "Colombia": (38100000, 82.3, 3.5),
    "France": (37930000, 60.4, 3.5),
    "Poland": (35310000, 92.2, 3.3),
    "Spain": (34670000, 75.2, 3.2),
    "Democratic Republic of the Congo": (31210000, 47.3, 2.9),
}
```

Here I used the apply_truth function to overwrite what had been scraped
using OCR. Then I looped it through the rows in the data frame.

``` python
def apply_truth(df, truth_dict):
    for idx, row in df.iterrows():
        country = row["Country"]
        if country in truth_dict:
            pop, pct_cat, pct_world = truth_dict[country]
            df.loc[idx, ["Catholic_Pop",
                         "Percent_Catholic",
                         "Percent_World_Catholics"]] = [pop, pct_cat, pct_world]
    return df

df_1910 = apply_truth(df_1910, truth_1910)
df_2010 = apply_truth(df_2010, truth_2010)
```

Here I printed out the clean dataframes

``` python
print("1910 clean:")
display(df_1910)

print("2010 clean:")
display(df_2010)
```

    1910 clean:

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country        | Year | Catholic_Pop | Percent_Catholic | Percent_World_Catholics |
|-----|----------------|------|--------------|------------------|-------------------------|
| 0   | France         | 1910 | 40510000     | 98.4             | 13.9                    |
| 1   | Italy          | 1910 | 35270000     | 99.9             | 12.1                    |
| 2   | Brazil         | 1910 | 21430000     | 95.6             | 7.4                     |
| 3   | Spain          | 1910 | 20350000     | 99.9             | 7.0                     |
| 4   | Poland         | 1910 | 18750000     | 77.1             | 6.4                     |
| 5   | Germany        | 1910 | 16580000     | 35.7             | 5.7                     |
| 6   | Mexico         | 1910 | 14280000     | 91.0             | 4.9                     |
| 7   | United States  | 1910 | 12470000     | 14.2             | 4.3                     |
| 8   | Philippines    | 1910 | 7260000      | 78.7             | 2.5                     |
| 9   | Czech Republic | 1910 | 7120000      | 86.2             | 2.4                     |

</div>

    2010 clean:

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|  | Country | Year | Catholic_Pop | Percent_Catholic | Percent_World_Catholics |
|----|----|----|----|----|----|
| 0 | Brazil | 2010 | 126750000 | 65.0 | 11.7 |
| 1 | Mexico | 2010 | 96450000 | 85.0 | 8.9 |
| 2 | Philippines | 2010 | 75570000 | 81.0 | 7.0 |
| 3 | United States | 2010 | 75380000 | 24.3 | 7.0 |
| 4 | Italy | 2010 | 49170000 | 81.2 | 4.6 |
| 5 | Colombia | 2010 | 38100000 | 82.3 | 3.5 |
| 6 | France | 2010 | 37930000 | 60.4 | 3.5 |
| 7 | Poland | 2010 | 35310000 | 92.2 | 3.3 |
| 8 | Spain | 2010 | 34670000 | 75.2 | 3.2 |
| 9 | Democratic Republic Of The Congo | 2010 | 34210000 | 473.0 | 20.0 |

</div>

# Population Growth Computations

## Now to compute population change growth numbers

Here I merged the dataframe to easily compute the growth factor and
percent change.

``` python
import pandas as pd

df_1910_small = df_1910[["Country", "Catholic_Pop"]].copy()
df_2010_small = df_2010[["Country", "Catholic_Pop"]].copy()

df_1910_small = df_1910_small.rename(columns={"Catholic_Pop": "Catholic_Pop_1910"})
df_2010_small = df_2010_small.rename(columns={"Catholic_Pop": "Catholic_Pop_2010"})

df_merge = df_1910_small.merge(df_2010_small, on="Country", how="inner")

df_merge["GrowthFactor"] = df_merge["Catholic_Pop_2010"] / df_merge["Catholic_Pop_1910"]
df_merge["Percent_Change"] = (
    (df_merge["Catholic_Pop_2010"] - df_merge["Catholic_Pop_1910"])
    / df_merge["Catholic_Pop_1910"]
)
df_merge
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|  | Country | Catholic_Pop_1910 | Catholic_Pop_2010 | GrowthFactor | Percent_Change |
|----|----|----|----|----|----|
| 0 | France | 40510000 | 37930000 | 0.936312 | -0.063688 |
| 1 | Italy | 35270000 | 49170000 | 1.394103 | 0.394103 |
| 2 | Brazil | 21430000 | 126750000 | 5.914606 | 4.914606 |
| 3 | Spain | 20350000 | 34670000 | 1.703686 | 0.703686 |
| 4 | Poland | 18750000 | 35310000 | 1.883200 | 0.883200 |
| 5 | Mexico | 14280000 | 96450000 | 6.754202 | 5.754202 |
| 6 | United States | 12470000 | 75380000 | 6.044908 | 5.044908 |
| 7 | Philippines | 7260000 | 75570000 | 10.409091 | 9.409091 |

</div>

Then i sorted by the greatest growth to see which country had the most
dramatic changes.

``` python
df_merge = df_merge.sort_values("GrowthFactor", ascending=False).reset_index(drop=True)

df_merge
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|  | Country | Catholic_Pop_1910 | Catholic_Pop_2010 | GrowthFactor | Percent_Change |
|----|----|----|----|----|----|
| 0 | Philippines | 7260000 | 75570000 | 10.409091 | 9.409091 |
| 1 | Mexico | 14280000 | 96450000 | 6.754202 | 5.754202 |
| 2 | United States | 12470000 | 75380000 | 6.044908 | 5.044908 |
| 3 | Brazil | 21430000 | 126750000 | 5.914606 | 4.914606 |
| 4 | Poland | 18750000 | 35310000 | 1.883200 | 0.883200 |
| 5 | Spain | 20350000 | 34670000 | 1.703686 | 0.703686 |
| 6 | Italy | 35270000 | 49170000 | 1.394103 | 0.394103 |
| 7 | France | 40510000 | 37930000 | 0.936312 | -0.063688 |

</div>

Lastly I made a bar chart visualizing the growth changes.

``` python
import matplotlib.pyplot as plt

plt.figure()
plt.bar(df_merge["Country"], df_merge["GrowthFactor"])
plt.xticks(rotation=45, ha="right")
plt.ylabel("Growth Factor (2010 / 1910)")
plt.title("Catholic Population Growth Factor by Country")
plt.tight_layout()
```

![](README_files/figure-commonmark/cell-18-output-1.png)

## Now to pull historic info from each country

Here I used beautiful soup again to pull information from different
wikipedia pages about the Catholic Church in those countries. I was
hoping that it would provide unbiased information compared to a news
source that might be biased politically or religiously. Additionally,
they have a verification system that removes any misinformation pretty
soon after posting, so this helped my reservations in using Wikipedia
for the project.

``` python
import requests
from bs4 import BeautifulSoup

countries = df_merge["Country"].tolist()

wiki_slugs = {
    "Philippines": "Catholic_Church_in_the_Philippines",
    "Mexico": "Catholic_Church_in_Mexico",
    "United States": "Catholic_Church_in_the_United_States",
    "Brazil": "Catholic_Church_in_Brazil",
    "Poland": "Catholic_Church_in_Poland",
    "Spain": "Catholic_Church_in_Spain",
    "Italy": "Catholic_Church_in_Italy",
    "France": "Catholic_Church_in_France",
}
```

Here I created a user-agent to access teh wikipedia pages to get past
potential blockages. Then I downloaded the webpages and parsed through
them to get the main text blocks. This block essentially defines the
scraping logic and how to go about it.

``` python
import requests
from bs4 import BeautifulSoup

headers = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/120.0.0.0 Safari/537.36"
    )
}

def scrape_country_text(country):
    slug = wiki_slugs[country]
    url = f"https://en.wikipedia.org/wiki/{slug}"

    resp = requests.get(url, headers=headers)
    resp.raise_for_status()  

    soup = BeautifulSoup(resp.text, "html.parser")

    content_div = soup.select_one("div.mw-parser-output")
    if content_div is None:
        return ""

    paragraphs = []

    for p in content_div.find_all("p", recursive=False):
        text = p.get_text(" ", strip=True)
        if text:
            paragraphs.append(text)

    if not paragraphs:
        for p in content_div.find_all("p"):
            text = p.get_text(" ", strip=True)
            if text:
                paragraphs.append(text)

    return " ".join(paragraphs)
```

Then I perform the actual scraping.

``` python
rows = []

for country in countries:
    print(f"Scraping {country}...")
    try:
        text = scrape_country_text(country)
    except Exception as e:
        print(f"  Error scraping {country}: {e}")
        text = ""  

    rows.append({"Country": country, "Text": text})

wiki_df = pd.DataFrame(rows)
print("Number of rows:", len(wiki_df))
display(wiki_df)
```

    Scraping Philippines...
    Scraping Mexico...
    Scraping United States...
    Scraping Brazil...
    Scraping Poland...
    Scraping Spain...
    Scraping Italy...
    Scraping France...
    Number of rows: 8

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Text                                              |
|-----|---------------|---------------------------------------------------|
| 0   | Philippines   | As part of the worldwide Catholic Church , the... |
| 1   | Mexico        | The Mexican Catholic Church , or Catholic Chur... |
| 2   | United States | The Catholic Church in the United States is pa... |
| 3   | Brazil        | The Brazilian Catholic Church , or Catholic Ch... |
| 4   | Poland        | Polish members of the Catholic Church , like e... |
| 5   | Spain         | The Spanish Catholic Church , or Catholic Chur... |
| 6   | Italy         | The Italian Catholic Church , or Catholic Chur... |
| 7   | France        | The Catholic Church in France , Gallican Churc... |

</div>

Now I’ll clean the text. I imported English_Stop_Words to get rid of
common words that might accidentally get picked up. I also changed
everything to all lowercase so it limits variation. Then I got rid of
puncuation, numbers and special characters. Then I tokenized it to get
it ready for topic modeling.

``` python
import re
from sklearn.feature_extraction.text import ENGLISH_STOP_WORDS

def clean_text(t):
    if not isinstance(t, str):
        return ""
    t = t.lower()
    t = re.sub(r"[^a-z\s]", " ", t)
    tokens = t.split()
    tokens = [w for w in tokens if len(w) > 2 and w not in ENGLISH_STOP_WORDS]
    return " ".join(tokens)

wiki_df["Clean_Text"] = wiki_df["Text"].apply(clean_text)
wiki_df[["Country", "Clean_Text"]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Clean_Text                                        |
|-----|---------------|---------------------------------------------------|
| 0   | Philippines   | worldwide catholic church catholic church phil... |
| 1   | Mexico        | mexican catholic church catholic church mexico... |
| 2   | United States | catholic church united states worldwide latin ... |
| 3   | Brazil        | brazilian catholic church catholic church braz... |
| 4   | Poland        | polish members catholic church like world spir... |
| 5   | Spain         | spanish catholic church catholic church spain ... |
| 6   | Italy         | italian catholic church catholic church italy ... |
| 7   | France        | catholic church france gallican church french ... |

</div>

# Topic Modeling

## Topic Modeling 1

Here I imported the necessary packages for vectorization. Then I split
the countries into 2 different growth groups. Then I vectorized the text
with various parameters. I ignored words that appeared in 95% of
documents and words that appeared in only one document. This way it
takes out super general words and allows for general themes to come
through.

``` python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.decomposition import LatentDirichletAllocation

high_growth = ["Philippines", "Mexico", "United States", "Brazil"]
low_growth  = ["France", "Italy", "Spain", "Poland"]

vectorizer = CountVectorizer(max_df=0.95, min_df=2)  
X = vectorizer.fit_transform(wiki_df["Clean_Text"])

feature_names = vectorizer.get_feature_names_out()
```

Here I defined the amount of topics I wanted. Then i set up the LDA
model. I told it to build 3 topics, and to have a random state of 42,
making it partly random and reproducible. I also told it to use
everything at once using batch.

``` python
n_topics = 3

lda = LatentDirichletAllocation(
    n_components=n_topics,
    random_state=42,
    learning_method="batch"
)
lda.fit(X)
```

<style>#sk-container-id-1 {
  /* Definition of color scheme common for light and dark mode */
  --sklearn-color-text: #000;
  --sklearn-color-text-muted: #666;
  --sklearn-color-line: gray;
  /* Definition of color scheme for unfitted estimators */
  --sklearn-color-unfitted-level-0: #fff5e6;
  --sklearn-color-unfitted-level-1: #f6e4d2;
  --sklearn-color-unfitted-level-2: #ffe0b3;
  --sklearn-color-unfitted-level-3: chocolate;
  /* Definition of color scheme for fitted estimators */
  --sklearn-color-fitted-level-0: #f0f8ff;
  --sklearn-color-fitted-level-1: #d4ebff;
  --sklearn-color-fitted-level-2: #b3dbfd;
  --sklearn-color-fitted-level-3: cornflowerblue;
&#10;  /* Specific color for light theme */
  --sklearn-color-text-on-default-background: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, black)));
  --sklearn-color-background: var(--sg-background-color, var(--theme-background, var(--jp-layout-color0, white)));
  --sklearn-color-border-box: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, black)));
  --sklearn-color-icon: #696969;
&#10;  @media (prefers-color-scheme: dark) {
    /* Redefinition of color scheme for dark theme */
    --sklearn-color-text-on-default-background: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, white)));
    --sklearn-color-background: var(--sg-background-color, var(--theme-background, var(--jp-layout-color0, #111)));
    --sklearn-color-border-box: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, white)));
    --sklearn-color-icon: #878787;
  }
}
&#10;#sk-container-id-1 {
  color: var(--sklearn-color-text);
}
&#10;#sk-container-id-1 pre {
  padding: 0;
}
&#10;#sk-container-id-1 input.sk-hidden--visually {
  border: 0;
  clip: rect(1px 1px 1px 1px);
  clip: rect(1px, 1px, 1px, 1px);
  height: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
  width: 1px;
}
&#10;#sk-container-id-1 div.sk-dashed-wrapped {
  border: 1px dashed var(--sklearn-color-line);
  margin: 0 0.4em 0.5em 0.4em;
  box-sizing: border-box;
  padding-bottom: 0.4em;
  background-color: var(--sklearn-color-background);
}
&#10;#sk-container-id-1 div.sk-container {
  /* jupyter's `normalize.less` sets `[hidden] { display: none; }`
     but bootstrap.min.css set `[hidden] { display: none !important; }`
     so we also need the `!important` here to be able to override the
     default hidden behavior on the sphinx rendered scikit-learn.org.
     See: https://github.com/scikit-learn/scikit-learn/issues/21755 */
  display: inline-block !important;
  position: relative;
}
&#10;#sk-container-id-1 div.sk-text-repr-fallback {
  display: none;
}
&#10;div.sk-parallel-item,
div.sk-serial,
div.sk-item {
  /* draw centered vertical line to link estimators */
  background-image: linear-gradient(var(--sklearn-color-text-on-default-background), var(--sklearn-color-text-on-default-background));
  background-size: 2px 100%;
  background-repeat: no-repeat;
  background-position: center center;
}
&#10;/* Parallel-specific style estimator block */
&#10;#sk-container-id-1 div.sk-parallel-item::after {
  content: "";
  width: 100%;
  border-bottom: 2px solid var(--sklearn-color-text-on-default-background);
  flex-grow: 1;
}
&#10;#sk-container-id-1 div.sk-parallel {
  display: flex;
  align-items: stretch;
  justify-content: center;
  background-color: var(--sklearn-color-background);
  position: relative;
}
&#10;#sk-container-id-1 div.sk-parallel-item {
  display: flex;
  flex-direction: column;
}
&#10;#sk-container-id-1 div.sk-parallel-item:first-child::after {
  align-self: flex-end;
  width: 50%;
}
&#10;#sk-container-id-1 div.sk-parallel-item:last-child::after {
  align-self: flex-start;
  width: 50%;
}
&#10;#sk-container-id-1 div.sk-parallel-item:only-child::after {
  width: 0;
}
&#10;/* Serial-specific style estimator block */
&#10;#sk-container-id-1 div.sk-serial {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: var(--sklearn-color-background);
  padding-right: 1em;
  padding-left: 1em;
}
&#10;
/* Toggleable style: style used for estimator/Pipeline/ColumnTransformer box that is
clickable and can be expanded/collapsed.
- Pipeline and ColumnTransformer use this feature and define the default style
- Estimators will overwrite some part of the style using the `sk-estimator` class
*/
&#10;/* Pipeline and ColumnTransformer style (default) */
&#10;#sk-container-id-1 div.sk-toggleable {
  /* Default theme specific background. It is overwritten whether we have a
  specific estimator or a Pipeline/ColumnTransformer */
  background-color: var(--sklearn-color-background);
}
&#10;/* Toggleable label */
#sk-container-id-1 label.sk-toggleable__label {
  cursor: pointer;
  display: flex;
  width: 100%;
  margin-bottom: 0;
  padding: 0.5em;
  box-sizing: border-box;
  text-align: center;
  align-items: start;
  justify-content: space-between;
  gap: 0.5em;
}
&#10;#sk-container-id-1 label.sk-toggleable__label .caption {
  font-size: 0.6rem;
  font-weight: lighter;
  color: var(--sklearn-color-text-muted);
}
&#10;#sk-container-id-1 label.sk-toggleable__label-arrow:before {
  /* Arrow on the left of the label */
  content: "▸";
  float: left;
  margin-right: 0.25em;
  color: var(--sklearn-color-icon);
}
&#10;#sk-container-id-1 label.sk-toggleable__label-arrow:hover:before {
  color: var(--sklearn-color-text);
}
&#10;/* Toggleable content - dropdown */
&#10;#sk-container-id-1 div.sk-toggleable__content {
  display: none;
  text-align: left;
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-1 div.sk-toggleable__content.fitted {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;#sk-container-id-1 div.sk-toggleable__content pre {
  margin: 0.2em;
  border-radius: 0.25em;
  color: var(--sklearn-color-text);
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-1 div.sk-toggleable__content.fitted pre {
  /* unfitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;#sk-container-id-1 input.sk-toggleable__control:checked~div.sk-toggleable__content {
  /* Expand drop-down */
  display: block;
  width: 100%;
  overflow: visible;
}
&#10;#sk-container-id-1 input.sk-toggleable__control:checked~label.sk-toggleable__label-arrow:before {
  content: "▾";
}
&#10;/* Pipeline/ColumnTransformer-specific style */
&#10;#sk-container-id-1 div.sk-label input.sk-toggleable__control:checked~label.sk-toggleable__label {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-1 div.sk-label.fitted input.sk-toggleable__control:checked~label.sk-toggleable__label {
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Estimator-specific style */
&#10;/* Colorize estimator box */
#sk-container-id-1 div.sk-estimator input.sk-toggleable__control:checked~label.sk-toggleable__label {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-1 div.sk-estimator.fitted input.sk-toggleable__control:checked~label.sk-toggleable__label {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;#sk-container-id-1 div.sk-label label.sk-toggleable__label,
#sk-container-id-1 div.sk-label label {
  /* The background is the default theme color */
  color: var(--sklearn-color-text-on-default-background);
}
&#10;/* On hover, darken the color of the background */
#sk-container-id-1 div.sk-label:hover label.sk-toggleable__label {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;/* Label box, darken color on hover, fitted */
#sk-container-id-1 div.sk-label.fitted:hover label.sk-toggleable__label.fitted {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Estimator label */
&#10;#sk-container-id-1 div.sk-label label {
  font-family: monospace;
  font-weight: bold;
  display: inline-block;
  line-height: 1.2em;
}
&#10;#sk-container-id-1 div.sk-label-container {
  text-align: center;
}
&#10;/* Estimator-specific */
#sk-container-id-1 div.sk-estimator {
  font-family: monospace;
  border: 1px dotted var(--sklearn-color-border-box);
  border-radius: 0.25em;
  box-sizing: border-box;
  margin-bottom: 0.5em;
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-1 div.sk-estimator.fitted {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;/* on hover */
#sk-container-id-1 div.sk-estimator:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-1 div.sk-estimator.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Specification for estimator info (e.g. "i" and "?") */
&#10;/* Common style for "i" and "?" */
&#10;.sk-estimator-doc-link,
a:link.sk-estimator-doc-link,
a:visited.sk-estimator-doc-link {
  float: right;
  font-size: smaller;
  line-height: 1em;
  font-family: monospace;
  background-color: var(--sklearn-color-background);
  border-radius: 1em;
  height: 1em;
  width: 1em;
  text-decoration: none !important;
  margin-left: 0.5em;
  text-align: center;
  /* unfitted */
  border: var(--sklearn-color-unfitted-level-1) 1pt solid;
  color: var(--sklearn-color-unfitted-level-1);
}
&#10;.sk-estimator-doc-link.fitted,
a:link.sk-estimator-doc-link.fitted,
a:visited.sk-estimator-doc-link.fitted {
  /* fitted */
  border: var(--sklearn-color-fitted-level-1) 1pt solid;
  color: var(--sklearn-color-fitted-level-1);
}
&#10;/* On hover */
div.sk-estimator:hover .sk-estimator-doc-link:hover,
.sk-estimator-doc-link:hover,
div.sk-label-container:hover .sk-estimator-doc-link:hover,
.sk-estimator-doc-link:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;div.sk-estimator.fitted:hover .sk-estimator-doc-link.fitted:hover,
.sk-estimator-doc-link.fitted:hover,
div.sk-label-container:hover .sk-estimator-doc-link.fitted:hover,
.sk-estimator-doc-link.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;/* Span, style for the box shown on hovering the info icon */
.sk-estimator-doc-link span {
  display: none;
  z-index: 9999;
  position: relative;
  font-weight: normal;
  right: .2ex;
  padding: .5ex;
  margin: .5ex;
  width: min-content;
  min-width: 20ex;
  max-width: 50ex;
  color: var(--sklearn-color-text);
  box-shadow: 2pt 2pt 4pt #999;
  /* unfitted */
  background: var(--sklearn-color-unfitted-level-0);
  border: .5pt solid var(--sklearn-color-unfitted-level-3);
}
&#10;.sk-estimator-doc-link.fitted span {
  /* fitted */
  background: var(--sklearn-color-fitted-level-0);
  border: var(--sklearn-color-fitted-level-3);
}
&#10;.sk-estimator-doc-link:hover span {
  display: block;
}
&#10;/* "?"-specific style due to the `<a>` HTML tag */
&#10;#sk-container-id-1 a.estimator_doc_link {
  float: right;
  font-size: 1rem;
  line-height: 1em;
  font-family: monospace;
  background-color: var(--sklearn-color-background);
  border-radius: 1rem;
  height: 1rem;
  width: 1rem;
  text-decoration: none;
  /* unfitted */
  color: var(--sklearn-color-unfitted-level-1);
  border: var(--sklearn-color-unfitted-level-1) 1pt solid;
}
&#10;#sk-container-id-1 a.estimator_doc_link.fitted {
  /* fitted */
  border: var(--sklearn-color-fitted-level-1) 1pt solid;
  color: var(--sklearn-color-fitted-level-1);
}
&#10;/* On hover */
#sk-container-id-1 a.estimator_doc_link:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;#sk-container-id-1 a.estimator_doc_link.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-3);
}
&#10;.estimator-table summary {
    padding: .5rem;
    font-family: monospace;
    cursor: pointer;
}
&#10;.estimator-table details[open] {
    padding-left: 0.1rem;
    padding-right: 0.1rem;
    padding-bottom: 0.3rem;
}
&#10;.estimator-table .parameters-table {
    margin-left: auto !important;
    margin-right: auto !important;
}
&#10;.estimator-table .parameters-table tr:nth-child(odd) {
    background-color: #fff;
}
&#10;.estimator-table .parameters-table tr:nth-child(even) {
    background-color: #f6f6f6;
}
&#10;.estimator-table .parameters-table tr:hover {
    background-color: #e0e0e0;
}
&#10;.estimator-table table td {
    border: 1px solid rgba(106, 105, 104, 0.232);
}
&#10;.user-set td {
    color:rgb(255, 94, 0);
    text-align: left;
}
&#10;.user-set td.value pre {
    color:rgb(255, 94, 0) !important;
    background-color: transparent !important;
}
&#10;.default td {
    color: black;
    text-align: left;
}
&#10;.user-set td i,
.default td i {
    color: black;
}
&#10;.copy-paste-icon {
    background-image: url(data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA0NDggNTEyIj48IS0tIUZvbnQgQXdlc29tZSBGcmVlIDYuNy4yIGJ5IEBmb250YXdlc29tZSAtIGh0dHBzOi8vZm9udGF3ZXNvbWUuY29tIExpY2Vuc2UgLSBodHRwczovL2ZvbnRhd2Vzb21lLmNvbS9saWNlbnNlL2ZyZWUgQ29weXJpZ2h0IDIwMjUgRm9udGljb25zLCBJbmMuLS0+PHBhdGggZD0iTTIwOCAwTDMzMi4xIDBjMTIuNyAwIDI0LjkgNS4xIDMzLjkgMTQuMWw2Ny45IDY3LjljOSA5IDE0LjEgMjEuMiAxNC4xIDMzLjlMNDQ4IDMzNmMwIDI2LjUtMjEuNSA0OC00OCA0OGwtMTkyIDBjLTI2LjUgMC00OC0yMS41LTQ4LTQ4bDAtMjg4YzAtMjYuNSAyMS41LTQ4IDQ4LTQ4ek00OCAxMjhsODAgMCAwIDY0LTY0IDAgMCAyNTYgMTkyIDAgMC0zMiA2NCAwIDAgNDhjMCAyNi41LTIxLjUgNDgtNDggNDhMNDggNTEyYy0yNi41IDAtNDgtMjEuNS00OC00OEwwIDE3NmMwLTI2LjUgMjEuNS00OCA0OC00OHoiLz48L3N2Zz4=);
    background-repeat: no-repeat;
    background-size: 14px 14px;
    background-position: 0;
    display: inline-block;
    width: 14px;
    height: 14px;
    cursor: pointer;
}
</style><body><div id="sk-container-id-1" class="sk-top-container"><div class="sk-text-repr-fallback"><pre>LatentDirichletAllocation(n_components=3, random_state=42)</pre><b>In a Jupyter environment, please rerun this cell to show the HTML representation or trust the notebook. <br />On GitHub, the HTML representation is unable to render, please try loading this page with nbviewer.org.</b></div><div class="sk-container" hidden><div class="sk-item"><div class="sk-estimator fitted sk-toggleable"><input class="sk-toggleable__control sk-hidden--visually" id="sk-estimator-id-1" type="checkbox" checked><label for="sk-estimator-id-1" class="sk-toggleable__label fitted sk-toggleable__label-arrow"><div><div>LatentDirichletAllocation</div></div><div><a class="sk-estimator-doc-link fitted" rel="noreferrer" target="_blank" href="https://scikit-learn.org/1.7/modules/generated/sklearn.decomposition.LatentDirichletAllocation.html">?<span>Documentation for LatentDirichletAllocation</span></a><span class="sk-estimator-doc-link fitted">i<span>Fitted</span></span></div></label><div class="sk-toggleable__content fitted" data-param-prefix="">
        <div class="estimator-table">
            <details>
                <summary>Parameters</summary>
                &#10;

|     |                      |           |
|-----|----------------------|-----------|
|     | n_components         | 3         |
|     | doc_topic_prior      | None      |
|     | topic_word_prior     | None      |
|     | learning_method      | 'batch'   |
|     | learning_decay       | 0.7       |
|     | learning_offset      | 10.0      |
|     | max_iter             | 10        |
|     | batch_size           | 128       |
|     | evaluate_every       | -1        |
|     | total_samples        | 1000000.0 |
|     | perp_tol             | 0.1       |
|     | mean_change_tol      | 0.001     |
|     | max_doc_update_iter  | 100       |
|     | n_jobs               | None      |
|     | verbose              | 0         |
|     | random_state         | 42        |

            </details>
        </div>
    </div></div></div></div></div><script>function copyToClipboard(text, element) {
    // Get the parameter prefix from the closest toggleable content
    const toggleableContent = element.closest('.sk-toggleable__content');
    const paramPrefix = toggleableContent ? toggleableContent.dataset.paramPrefix : '';
    const fullParamName = paramPrefix ? `${paramPrefix}${text}` : text;
&#10;    const originalStyle = element.style;
    const computedStyle = window.getComputedStyle(element);
    const originalWidth = computedStyle.width;
    const originalHTML = element.innerHTML.replace('Copied!', '');
&#10;    navigator.clipboard.writeText(fullParamName)
        .then(() => {
            element.style.width = originalWidth;
            element.style.color = 'green';
            element.innerHTML = "Copied!";
&#10;            setTimeout(() => {
                element.innerHTML = originalHTML;
                element.style = originalStyle;
            }, 2000);
        })
        .catch(err => {
            console.error('Failed to copy:', err);
            element.style.color = 'red';
            element.innerHTML = "Failed!";
            setTimeout(() => {
                element.innerHTML = originalHTML;
                element.style = originalStyle;
            }, 2000);
        });
    return false;
}
&#10;document.querySelectorAll('.fa-regular.fa-copy').forEach(function(element) {
    const toggleableContent = element.closest('.sk-toggleable__content');
    const paramPrefix = toggleableContent ? toggleableContent.dataset.paramPrefix : '';
    const paramName = element.parentElement.nextElementSibling.textContent.trim();
    const fullParamName = paramPrefix ? `${paramPrefix}${paramName}` : paramName;
&#10;    element.setAttribute('title', fullParamName);
});
</script></body>

Here I printed out the top words for each topic.

``` python
def print_top_words(model, feature_names, n_top_words=10):
    for topic_idx, topic in enumerate(model.components_):
        top_indices = topic.argsort()[:-n_top_words - 1:-1]
        top_words = [feature_names[i] for i in top_indices]
        print(f"Topic #{topic_idx}: {' | '.join(top_words)}")

print_top_words(lda, feature_names, n_top_words=10)
```

    Topic #0: population | brazil | identified | catholics | census | mexico | million | country | survey | national
    Topic #1: rome | cathedral | saint | communion | dioceses | bishop | century | peter | city | catholics
    Topic #2: spanish | population | largest | state | philippines | conference | united | world | states | period

### Topic Distribution

Here I showed the way that each topic was distributed among the
countries. I was hoping to see some patterns among high growth and low
growth. It appears that low growth countries really liked topic 1 (2),
which the high growth countries really liked topics 0 and 2 (1&3)

``` python
doc_topic_dist = lda.transform(X)  

for i in range(n_topics):
    wiki_df[f"Topic_{i}_prob"] = doc_topic_dist[:, i]

wiki_df[["Country"] + [f"Topic_{i}_prob" for i in range(n_topics)]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Topic_0_prob | Topic_1_prob | Topic_2_prob |
|-----|---------------|--------------|--------------|--------------|
| 0   | Philippines   | 0.008866     | 0.008337     | 0.982797     |
| 1   | Mexico        | 0.976034     | 0.010697     | 0.013269     |
| 2   | United States | 0.016021     | 0.015512     | 0.968467     |
| 3   | Brazil        | 0.989767     | 0.005050     | 0.005183     |
| 4   | Poland        | 0.985443     | 0.007306     | 0.007251     |
| 5   | Spain         | 0.022123     | 0.022432     | 0.955445     |
| 6   | Italy         | 0.007853     | 0.984196     | 0.007950     |
| 7   | France        | 0.005545     | 0.989015     | 0.005440     |

</div>

Here I continued with that analysis and calculated average topic
probabilities. This reinforced the same topic finding associations from
above.

``` python
wiki_df["Growth_Group"] = wiki_df["Country"].apply(
    lambda c: "High" if c in high_growth else "Low"
)

topic_cols = [f"Topic_{i}_prob" for i in range(n_topics)]

group_topics = wiki_df.groupby("Growth_Group")[topic_cols].mean()
group_topics
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|              | Topic_0_prob | Topic_1_prob | Topic_2_prob |
|--------------|--------------|--------------|--------------|
| Growth_Group |              |              |              |
| High         | 0.497672     | 0.009899     | 0.492429     |
| Low          | 0.255241     | 0.500737     | 0.244022     |

</div>

## Topic Modeling 2

I wanted to try and continue to find patterns with topic modeling. So I
did high growth key words versus low growth key words.

``` python
from sklearn.feature_extraction.text import TfidfVectorizer
import pandas as pd
import numpy as np

wiki_df["Growth_Group"] = wiki_df["Country"].apply(
    lambda c: "High" if c in high_growth else "Low"
)

group_texts = wiki_df.groupby("Growth_Group")["Clean_Text"].apply(lambda x: " ".join(x))

vectorizer = TfidfVectorizer(max_features=2000)
tfidf_matrix = vectorizer.fit_transform(group_texts)

feature_names = np.array(vectorizer.get_feature_names_out())

tfidf_df = pd.DataFrame(
    tfidf_matrix.toarray(),
    index=group_texts.index,
    columns=feature_names
)

top_high = tfidf_df.loc["High"].sort_values(ascending=False).head(20)
top_low = tfidf_df.loc["Low"].sort_values(ascending=False).head(20)

print("Top High-Growth Keywords:\n", top_high)
print("\nTop Low-Growth Keywords:\n", top_low)
```

    Top High-Growth Keywords:
     catholic       0.582849
    church         0.376031
    brazil         0.290674
    population     0.244420
    mexico         0.184975
    brazilian      0.158550
    largest        0.158550
    identified     0.112809
    philippines    0.105700
    period         0.105700
    faith          0.105700
    country        0.094008
    according      0.079275
    philippine     0.079275
    percent        0.079275
    mexican        0.079275
    census         0.075206
    world          0.075206
    pope           0.075206
    million        0.075206
    Name: High, dtype: float64

    Top Low-Growth Keywords:
     church       0.490928
    catholic     0.457071
    france       0.190340
    rome         0.186214
    cathedral    0.166547
    french       0.166547
    italy        0.142755
    italian      0.142755
    poland       0.142755
    saint        0.118962
    pope         0.118500
    catholics    0.101571
    bishop       0.095170
    dioceses     0.084643
    countries    0.071377
    city         0.071377
    includes     0.071377
    paris        0.071377
    priestly     0.071377
    baptized     0.071377
    Name: Low, dtype: float64

### High growth topics versus Low Growth Topics.

It didn’t give much more insight.

``` python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.decomposition import LatentDirichletAllocation

def lda_for_group(group_name):
    texts = wiki_df[wiki_df["Growth_Group"] == group_name]["Clean_Text"].tolist()
    vectorizer = CountVectorizer(max_df=0.95, min_df=1)
    X = vectorizer.fit_transform(texts)
    lda = LatentDirichletAllocation(n_components=2, random_state=42)
    lda.fit(X)
    return lda, vectorizer.get_feature_names_out()

def print_topics(model, feature_names, n_top=12):
    for idx, topic in enumerate(model.components_):
        top_indices = topic.argsort()[:-n_top - 1:-1]
        words = [feature_names[i] for i in top_indices]
        print(f"Topic #{idx}: {' | '.join(words)}\n")

lda_high, feat_high = lda_for_group("High")
print("\n=== HIGH-GROWTH TOPICS ===")
print_topics(lda_high, feat_high)

lda_low, feat_low = lda_for_group("Low")
print("\n=== LOW-GROWTH TOPICS ===")
print_topics(lda_low, feat_low)
```


    === HIGH-GROWTH TOPICS ===
    Topic #0: largest | faith | philippines | world | philippine | united | states | spanish | catholicism | conference | period | state

    Topic #1: brazil | identified | brazilian | mexico | census | million | catholics | national | according | mexican | percent | bishops


    === LOW-GROWTH TOPICS ===
    Topic #0: poland | spanish | catholics | includes | conference | baptized | population | catholicism | countries | state | survey | comprise

    Topic #1: france | french | cathedral | italy | italian | saint | dioceses | communion | bishop | priestly | paris | century

## Topic modeling (attempt 3:(stricter))

I had felt like The words from the first rounds of topic modeling above
weren’t very helpful. So I redid it with many stopwords that somewhat
helped to bring forth more unique and characteristic words. I was really
just hoping to bring about stronger insights. I also narrowed it down to
just 2 topics in hopes of having a clearer distinction between the
themes that emerge. I realized I accidentally deleted a crucial code
block with the “clean text” column so I added it back in here, before
going through with the new topic modeling.

``` python
import re
from sklearn.feature_extraction.text import ENGLISH_STOP_WORDS

def clean_text(t):
    if not isinstance(t, str):
        return ""
    t = t.lower()
    t = re.sub(r"[^a-z\s]", " ", t)
    tokens = t.split()
    tokens = [w for w in tokens if len(w) > 2 and w not in ENGLISH_STOP_WORDS]
    return " ".join(tokens)

wiki_df["Clean_Text"] = wiki_df["Text"].apply(clean_text)
```

new topic modeling method

``` python
from sklearn.feature_extraction.text import CountVectorizer, ENGLISH_STOP_WORDS
from sklearn.decomposition import LatentDirichletAllocation


extra_stopwords = {
    "population", "brazil", "mexico", "spanish", "philippines", "religious", "approximately",
    "united", "states", "catholics", "largest", "country", "church", "catholic",
    "census", "million", "survey", "national",
    "rome", "cathedral", "dioceses", "bishop",
    "century", "peter", "city", "conference",
    "world", "state", "period", "identified",
    "christian", "catholicism", "members", "number", "data", "worldwide", "countries",
    "according", "bishops", "statistics", "spiritual", "leadership", "holy", "previous", "estimated", "having", "communion", "latin", "second", "episcopal", "society", "includes", "year", "beginning", "early", "established", "served",
    "nations", "primate", "eparchies", "surveyed", "adherents"}

custom_stop = list(ENGLISH_STOP_WORDS.union(extra_stopwords))

vectorizer = CountVectorizer(
    max_df=0.95,
    min_df=2,
    stop_words=custom_stop,
    ngram_range=(1, 2) 
)

X = vectorizer.fit_transform(wiki_df["Clean_Text"])
feature_names = vectorizer.get_feature_names_out()

n_topics = 2

lda = LatentDirichletAllocation(
    n_components=n_topics,
    random_state=42,
    learning_method="batch"
)
lda.fit(X)

def print_top_words(model, feature_names, n_top_words=10):
    for topic_idx, topic in enumerate(model.components_):
        top_indices = topic.argsort()[:-n_top_words - 1:-1]
        top_words = [feature_names[i] for i in top_indices]
        print(f"Topic #{topic_idx}: {' | '.join(top_words)}")

print_top_words(lda, feature_names, n_top_words=10)
```

    Topic #0: percent | religion | christianity | baptized | roman | jurisdictions | eastern | practiced | archbishop | mass
    Topic #1: saint | official | vatican | empire | roman | roman empire | saints | including | pilgrimage | paul

### Topic Distribution

Here I ran the topic distribution again to see how it may have been
affected by the stop words.

``` python
doc_topic_dist = lda.transform(X)  

for i in range(n_topics):
    wiki_df[f"Topic_{i}_prob"] = doc_topic_dist[:, i]

wiki_df[["Country"] + [f"Topic_{i}_prob" for i in range(n_topics)]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Topic_0_prob | Topic_1_prob |
|-----|---------------|--------------|--------------|
| 0   | Philippines   | 0.888282     | 0.111718     |
| 1   | Mexico        | 0.944775     | 0.055225     |
| 2   | United States | 0.747917     | 0.252083     |
| 3   | Brazil        | 0.085807     | 0.914193     |
| 4   | Poland        | 0.941373     | 0.058627     |
| 5   | Spain         | 0.500000     | 0.500000     |
| 6   | Italy         | 0.035767     | 0.964233     |
| 7   | France        | 0.028431     | 0.971569     |

</div>

Here I split it into high growth and low growth again. I found that
topic 0 is more heavily leaned into by the high growth group. While the
low growth group more heavily leans into Topic 1. It seems as if topic 0
has themes of evangelization, while topic 1 has themes of historical
influence. Which would correlate with findings of growth vs decay.

``` python
wiki_df["Growth_Group"] = wiki_df["Country"].apply(
    lambda c: "High" if c in high_growth else "Low"
)

topic_cols = [f"Topic_{i}_prob" for i in range(n_topics)]

group_topics = wiki_df.groupby("Growth_Group")[topic_cols].mean()
group_topics
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|              | Topic_0_prob | Topic_1_prob |
|--------------|--------------|--------------|
| Growth_Group |              |              |
| High         | 0.666695     | 0.333305     |
| Low          | 0.376393     | 0.623607     |

</div>

# Sentiment Analysis

## Sentiment Analysis Score per country (1)

Next I was curious to do a sentiment analysis, because why not? Actually
I was curious if certain words in alignment with growth versus decay
might show up here. I created positive and negative sets of words.

``` python
positive_words = {
    "growth", "expansion", "revival", "mission", "missions",
    "evangelization", "conversion", "conversions", "renewal",
    "recognition", "rights", "freedom", "liberty"
}

negative_words = {
    "decline", "secularization", "persecution", "persecutions",
    "repression", "scandal", "scandals", "crisis", "crises",
    "conflict", "war", "wars", "suppression", "anti", "hostility"
}
```

Then I did the actual analysis. I split the text into individual token
words, separately counted how many were in the positive and negative
lists, and then created a score between -1 and 1. Unfortunately, only
negative words got picked up on. So there wasn’t much insight here.

``` python
def simple_sentiment_score(text):
    if not isinstance(text, str):
        return 0.0
    tokens = text.split()
    if not tokens:
        return 0.0

    pos = sum(1 for t in tokens if t in positive_words)
    neg = sum(1 for t in tokens if t in negative_words)

    return (pos - neg) / len(tokens)

wiki_df["Sentiment_Score"] = wiki_df["Clean_Text"].apply(simple_sentiment_score)

wiki_df[["Country", "Sentiment_Score"]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Sentiment_Score |
|-----|---------------|-----------------|
| 0   | Philippines   | 0.000000        |
| 1   | Mexico        | 0.000000        |
| 2   | United States | 0.000000        |
| 3   | Brazil        | 0.000000        |
| 4   | Poland        | -0.008696       |
| 5   | Spain         | 0.000000        |
| 6   | Italy         | 0.000000        |
| 7   | France        | -0.007463       |

</div>

I split the sentiment score by growth group, but it didn’t really give
anymore information.

``` python
sent_group = wiki_df.groupby("Growth_Group")["Sentiment_Score"].mean()
sent_group
```

    Growth_Group
    High    0.00000
    Low    -0.00404
    Name: Sentiment_Score, dtype: float64

Next I made a bar chart showing the sentiment analysis.

``` python
plt.figure()
plt.bar(wiki_df["Country"], wiki_df["Sentiment_Score"])
plt.xticks(rotation=45, ha="right")
plt.ylabel("Sentiment Score (simple lexicon)")
plt.title("Sentiment of Catholic History Text by Country")
plt.tight_layout()
```

![](README_files/figure-commonmark/cell-37-output-1.png)

## Attempting supervised machine learning sentiment analysis (2)

I wanted to try a different form of sentiment analysis without my own
chosen words. Here I used regex to break the wikipedia pages down by
sentences.

``` python
def split_into_sentences(text):
    import re
    if not isinstance(text, str):
        return []
    text = re.sub(r'([.!?])', r'\1 ', text)
    sentences = re.split(r'(?<=[.!?])\s+', text)
    return [s.strip() for s in sentences if s.strip()]
```

Next i went through each countries Wikipedia pages and collected the
sentences into a list. I also made sure to only get sentences longer
than 6 words in order to avoid fragments and titles.

``` python
all_sentences = []

for text in wiki_df["Text"]:
    for s in split_into_sentences(text):
        if len(s.split()) > 6:  
            all_sentences.append(s)
```

Here I imported a set of randomly shuffled sentences that I will later
manually label.

``` python
import random

random.shuffle(all_sentences)
sample_sentences = all_sentences[:120]   
```

Here I created a data frame that will allow me to store their sentiment
labels.

``` python
import pandas as pd

df_label = pd.DataFrame({
    "sentence": sample_sentences,
    "label": [""] * len(sample_sentences)  
})

df_label.head(10)
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | sentence                                            | label |
|-----|-----------------------------------------------------|-------|
| 0   | The pope resides in Vatican City, enclaved in ...   |       |
| 1   | \[ 8 \] The capital city, Paris , is a major pil... |       |
| 2   | 3% of the population identified themselves as ...   |       |
| 3   | According to the Mexican census, Roman Catholi...   |       |
| 4   | Established in the second century in unbroken ...   |       |
| 5   | During the same period, Protestants increased ...   |       |
| 6   | Combined, these comprise about 10,000 parishes...   |       |
| 7   | 7 percent of the population in 2020.                |       |
| 8   | Notable churches of France include Notre Dame ...   |       |
| 9   | \[ 7 \] 80 to 90 priests are ordained every year... |       |

</div>

Next I exported the the table into a CSV file.

``` python
df_label.to_csv("labeling_sentences.csv", index=False)
```

Here I manually labeled 83 rows of data (And saved as a different file
name so that if the above block is rerun, the labels wont be erased).
Unforunately learned that one the hard way when running through my code
and accidentally erased all my labels :\_(

``` python
df_label = pd.read_csv("labeling_sentences1.csv")
df_label
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | sentence                                          | label    |
|-----|---------------------------------------------------|----------|
| 0   | The Brazilian Catholic Church , or Catholic Ch... | positive |
| 1   | The pope resides in Vatican City, enclaved in ... | neutral  |
| 2   | Peter , Institute of Christ the King Sovereign... | negative |
| 3   | Having been a major centre for Christian pilgr... | positive |
| 4   | The status of the Catholic Church as the sole ... | positive |
| ... | ...                                               | ...      |
| 77  | In 496 Remigius baptized King Clovis I , who t... | positive |
| 78  | Its national shrine , Lourdes , is visited by ... | positive |
| 79  | Established in the second century in unbroken ... | positive |
| 80  | 3% of the population identified themselves as ... | negative |
| 81  | Mark 's in Venice , and Brunelleschi 's Floren... | neutral  |

<p>82 rows × 2 columns</p>
</div>

Here I dropped rows with neutral labels. Then I counted how many rows
are positive or negative.

``` python
df_label = df_label[df_label["label"] != "neutral"].reset_index(drop=True)
df_label["label"].value_counts()
```

    label
    positive    31
    negative    15
    Name: count, dtype: int64

Here I imported the Machine Learning packages I would need to create the
pipeline.

``` python
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report, accuracy_score
import pandas as pd
```

Here I defined my training data(75%) and testing data(25%). I also
stratified it to make sure that there was a fair representation of
labels among the data.

``` python
X_train, X_test, y_train, y_test = train_test_split(
    df_label["sentence"],
    df_label["label"],
    test_size=0.25,
    random_state=42,
    stratify=df_label["label"]
)
```

Next I vectorized the data by converting the sentences to numeric
features that captured the importance of the words. I integrated stop
words and limited the vocabulary to 5000.

``` python
tfidf = TfidfVectorizer(
    stop_words="english",
    max_features=5000,
    ngram_range=(1,2)   
)

X_train_vec = tfidf.fit_transform(X_train)
X_test_vec = tfidf.transform(X_test)
```

Here I ran the logistic regression. I kept the max iteration to 300 to
give it enough to converge but not too much to make it take forever.

``` python
clf = LogisticRegression(max_iter=300)
clf.fit(X_train_vec, y_train)
```

<style>#sk-container-id-2 {
  /* Definition of color scheme common for light and dark mode */
  --sklearn-color-text: #000;
  --sklearn-color-text-muted: #666;
  --sklearn-color-line: gray;
  /* Definition of color scheme for unfitted estimators */
  --sklearn-color-unfitted-level-0: #fff5e6;
  --sklearn-color-unfitted-level-1: #f6e4d2;
  --sklearn-color-unfitted-level-2: #ffe0b3;
  --sklearn-color-unfitted-level-3: chocolate;
  /* Definition of color scheme for fitted estimators */
  --sklearn-color-fitted-level-0: #f0f8ff;
  --sklearn-color-fitted-level-1: #d4ebff;
  --sklearn-color-fitted-level-2: #b3dbfd;
  --sklearn-color-fitted-level-3: cornflowerblue;
&#10;  /* Specific color for light theme */
  --sklearn-color-text-on-default-background: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, black)));
  --sklearn-color-background: var(--sg-background-color, var(--theme-background, var(--jp-layout-color0, white)));
  --sklearn-color-border-box: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, black)));
  --sklearn-color-icon: #696969;
&#10;  @media (prefers-color-scheme: dark) {
    /* Redefinition of color scheme for dark theme */
    --sklearn-color-text-on-default-background: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, white)));
    --sklearn-color-background: var(--sg-background-color, var(--theme-background, var(--jp-layout-color0, #111)));
    --sklearn-color-border-box: var(--sg-text-color, var(--theme-code-foreground, var(--jp-content-font-color1, white)));
    --sklearn-color-icon: #878787;
  }
}
&#10;#sk-container-id-2 {
  color: var(--sklearn-color-text);
}
&#10;#sk-container-id-2 pre {
  padding: 0;
}
&#10;#sk-container-id-2 input.sk-hidden--visually {
  border: 0;
  clip: rect(1px 1px 1px 1px);
  clip: rect(1px, 1px, 1px, 1px);
  height: 1px;
  margin: -1px;
  overflow: hidden;
  padding: 0;
  position: absolute;
  width: 1px;
}
&#10;#sk-container-id-2 div.sk-dashed-wrapped {
  border: 1px dashed var(--sklearn-color-line);
  margin: 0 0.4em 0.5em 0.4em;
  box-sizing: border-box;
  padding-bottom: 0.4em;
  background-color: var(--sklearn-color-background);
}
&#10;#sk-container-id-2 div.sk-container {
  /* jupyter's `normalize.less` sets `[hidden] { display: none; }`
     but bootstrap.min.css set `[hidden] { display: none !important; }`
     so we also need the `!important` here to be able to override the
     default hidden behavior on the sphinx rendered scikit-learn.org.
     See: https://github.com/scikit-learn/scikit-learn/issues/21755 */
  display: inline-block !important;
  position: relative;
}
&#10;#sk-container-id-2 div.sk-text-repr-fallback {
  display: none;
}
&#10;div.sk-parallel-item,
div.sk-serial,
div.sk-item {
  /* draw centered vertical line to link estimators */
  background-image: linear-gradient(var(--sklearn-color-text-on-default-background), var(--sklearn-color-text-on-default-background));
  background-size: 2px 100%;
  background-repeat: no-repeat;
  background-position: center center;
}
&#10;/* Parallel-specific style estimator block */
&#10;#sk-container-id-2 div.sk-parallel-item::after {
  content: "";
  width: 100%;
  border-bottom: 2px solid var(--sklearn-color-text-on-default-background);
  flex-grow: 1;
}
&#10;#sk-container-id-2 div.sk-parallel {
  display: flex;
  align-items: stretch;
  justify-content: center;
  background-color: var(--sklearn-color-background);
  position: relative;
}
&#10;#sk-container-id-2 div.sk-parallel-item {
  display: flex;
  flex-direction: column;
}
&#10;#sk-container-id-2 div.sk-parallel-item:first-child::after {
  align-self: flex-end;
  width: 50%;
}
&#10;#sk-container-id-2 div.sk-parallel-item:last-child::after {
  align-self: flex-start;
  width: 50%;
}
&#10;#sk-container-id-2 div.sk-parallel-item:only-child::after {
  width: 0;
}
&#10;/* Serial-specific style estimator block */
&#10;#sk-container-id-2 div.sk-serial {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: var(--sklearn-color-background);
  padding-right: 1em;
  padding-left: 1em;
}
&#10;
/* Toggleable style: style used for estimator/Pipeline/ColumnTransformer box that is
clickable and can be expanded/collapsed.
- Pipeline and ColumnTransformer use this feature and define the default style
- Estimators will overwrite some part of the style using the `sk-estimator` class
*/
&#10;/* Pipeline and ColumnTransformer style (default) */
&#10;#sk-container-id-2 div.sk-toggleable {
  /* Default theme specific background. It is overwritten whether we have a
  specific estimator or a Pipeline/ColumnTransformer */
  background-color: var(--sklearn-color-background);
}
&#10;/* Toggleable label */
#sk-container-id-2 label.sk-toggleable__label {
  cursor: pointer;
  display: flex;
  width: 100%;
  margin-bottom: 0;
  padding: 0.5em;
  box-sizing: border-box;
  text-align: center;
  align-items: start;
  justify-content: space-between;
  gap: 0.5em;
}
&#10;#sk-container-id-2 label.sk-toggleable__label .caption {
  font-size: 0.6rem;
  font-weight: lighter;
  color: var(--sklearn-color-text-muted);
}
&#10;#sk-container-id-2 label.sk-toggleable__label-arrow:before {
  /* Arrow on the left of the label */
  content: "▸";
  float: left;
  margin-right: 0.25em;
  color: var(--sklearn-color-icon);
}
&#10;#sk-container-id-2 label.sk-toggleable__label-arrow:hover:before {
  color: var(--sklearn-color-text);
}
&#10;/* Toggleable content - dropdown */
&#10;#sk-container-id-2 div.sk-toggleable__content {
  display: none;
  text-align: left;
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-2 div.sk-toggleable__content.fitted {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;#sk-container-id-2 div.sk-toggleable__content pre {
  margin: 0.2em;
  border-radius: 0.25em;
  color: var(--sklearn-color-text);
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-2 div.sk-toggleable__content.fitted pre {
  /* unfitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;#sk-container-id-2 input.sk-toggleable__control:checked~div.sk-toggleable__content {
  /* Expand drop-down */
  display: block;
  width: 100%;
  overflow: visible;
}
&#10;#sk-container-id-2 input.sk-toggleable__control:checked~label.sk-toggleable__label-arrow:before {
  content: "▾";
}
&#10;/* Pipeline/ColumnTransformer-specific style */
&#10;#sk-container-id-2 div.sk-label input.sk-toggleable__control:checked~label.sk-toggleable__label {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-2 div.sk-label.fitted input.sk-toggleable__control:checked~label.sk-toggleable__label {
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Estimator-specific style */
&#10;/* Colorize estimator box */
#sk-container-id-2 div.sk-estimator input.sk-toggleable__control:checked~label.sk-toggleable__label {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-2 div.sk-estimator.fitted input.sk-toggleable__control:checked~label.sk-toggleable__label {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;#sk-container-id-2 div.sk-label label.sk-toggleable__label,
#sk-container-id-2 div.sk-label label {
  /* The background is the default theme color */
  color: var(--sklearn-color-text-on-default-background);
}
&#10;/* On hover, darken the color of the background */
#sk-container-id-2 div.sk-label:hover label.sk-toggleable__label {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;/* Label box, darken color on hover, fitted */
#sk-container-id-2 div.sk-label.fitted:hover label.sk-toggleable__label.fitted {
  color: var(--sklearn-color-text);
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Estimator label */
&#10;#sk-container-id-2 div.sk-label label {
  font-family: monospace;
  font-weight: bold;
  display: inline-block;
  line-height: 1.2em;
}
&#10;#sk-container-id-2 div.sk-label-container {
  text-align: center;
}
&#10;/* Estimator-specific */
#sk-container-id-2 div.sk-estimator {
  font-family: monospace;
  border: 1px dotted var(--sklearn-color-border-box);
  border-radius: 0.25em;
  box-sizing: border-box;
  margin-bottom: 0.5em;
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-0);
}
&#10;#sk-container-id-2 div.sk-estimator.fitted {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-0);
}
&#10;/* on hover */
#sk-container-id-2 div.sk-estimator:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-2);
}
&#10;#sk-container-id-2 div.sk-estimator.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-2);
}
&#10;/* Specification for estimator info (e.g. "i" and "?") */
&#10;/* Common style for "i" and "?" */
&#10;.sk-estimator-doc-link,
a:link.sk-estimator-doc-link,
a:visited.sk-estimator-doc-link {
  float: right;
  font-size: smaller;
  line-height: 1em;
  font-family: monospace;
  background-color: var(--sklearn-color-background);
  border-radius: 1em;
  height: 1em;
  width: 1em;
  text-decoration: none !important;
  margin-left: 0.5em;
  text-align: center;
  /* unfitted */
  border: var(--sklearn-color-unfitted-level-1) 1pt solid;
  color: var(--sklearn-color-unfitted-level-1);
}
&#10;.sk-estimator-doc-link.fitted,
a:link.sk-estimator-doc-link.fitted,
a:visited.sk-estimator-doc-link.fitted {
  /* fitted */
  border: var(--sklearn-color-fitted-level-1) 1pt solid;
  color: var(--sklearn-color-fitted-level-1);
}
&#10;/* On hover */
div.sk-estimator:hover .sk-estimator-doc-link:hover,
.sk-estimator-doc-link:hover,
div.sk-label-container:hover .sk-estimator-doc-link:hover,
.sk-estimator-doc-link:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;div.sk-estimator.fitted:hover .sk-estimator-doc-link.fitted:hover,
.sk-estimator-doc-link.fitted:hover,
div.sk-label-container:hover .sk-estimator-doc-link.fitted:hover,
.sk-estimator-doc-link.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;/* Span, style for the box shown on hovering the info icon */
.sk-estimator-doc-link span {
  display: none;
  z-index: 9999;
  position: relative;
  font-weight: normal;
  right: .2ex;
  padding: .5ex;
  margin: .5ex;
  width: min-content;
  min-width: 20ex;
  max-width: 50ex;
  color: var(--sklearn-color-text);
  box-shadow: 2pt 2pt 4pt #999;
  /* unfitted */
  background: var(--sklearn-color-unfitted-level-0);
  border: .5pt solid var(--sklearn-color-unfitted-level-3);
}
&#10;.sk-estimator-doc-link.fitted span {
  /* fitted */
  background: var(--sklearn-color-fitted-level-0);
  border: var(--sklearn-color-fitted-level-3);
}
&#10;.sk-estimator-doc-link:hover span {
  display: block;
}
&#10;/* "?"-specific style due to the `<a>` HTML tag */
&#10;#sk-container-id-2 a.estimator_doc_link {
  float: right;
  font-size: 1rem;
  line-height: 1em;
  font-family: monospace;
  background-color: var(--sklearn-color-background);
  border-radius: 1rem;
  height: 1rem;
  width: 1rem;
  text-decoration: none;
  /* unfitted */
  color: var(--sklearn-color-unfitted-level-1);
  border: var(--sklearn-color-unfitted-level-1) 1pt solid;
}
&#10;#sk-container-id-2 a.estimator_doc_link.fitted {
  /* fitted */
  border: var(--sklearn-color-fitted-level-1) 1pt solid;
  color: var(--sklearn-color-fitted-level-1);
}
&#10;/* On hover */
#sk-container-id-2 a.estimator_doc_link:hover {
  /* unfitted */
  background-color: var(--sklearn-color-unfitted-level-3);
  color: var(--sklearn-color-background);
  text-decoration: none;
}
&#10;#sk-container-id-2 a.estimator_doc_link.fitted:hover {
  /* fitted */
  background-color: var(--sklearn-color-fitted-level-3);
}
&#10;.estimator-table summary {
    padding: .5rem;
    font-family: monospace;
    cursor: pointer;
}
&#10;.estimator-table details[open] {
    padding-left: 0.1rem;
    padding-right: 0.1rem;
    padding-bottom: 0.3rem;
}
&#10;.estimator-table .parameters-table {
    margin-left: auto !important;
    margin-right: auto !important;
}
&#10;.estimator-table .parameters-table tr:nth-child(odd) {
    background-color: #fff;
}
&#10;.estimator-table .parameters-table tr:nth-child(even) {
    background-color: #f6f6f6;
}
&#10;.estimator-table .parameters-table tr:hover {
    background-color: #e0e0e0;
}
&#10;.estimator-table table td {
    border: 1px solid rgba(106, 105, 104, 0.232);
}
&#10;.user-set td {
    color:rgb(255, 94, 0);
    text-align: left;
}
&#10;.user-set td.value pre {
    color:rgb(255, 94, 0) !important;
    background-color: transparent !important;
}
&#10;.default td {
    color: black;
    text-align: left;
}
&#10;.user-set td i,
.default td i {
    color: black;
}
&#10;.copy-paste-icon {
    background-image: url(data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA0NDggNTEyIj48IS0tIUZvbnQgQXdlc29tZSBGcmVlIDYuNy4yIGJ5IEBmb250YXdlc29tZSAtIGh0dHBzOi8vZm9udGF3ZXNvbWUuY29tIExpY2Vuc2UgLSBodHRwczovL2ZvbnRhd2Vzb21lLmNvbS9saWNlbnNlL2ZyZWUgQ29weXJpZ2h0IDIwMjUgRm9udGljb25zLCBJbmMuLS0+PHBhdGggZD0iTTIwOCAwTDMzMi4xIDBjMTIuNyAwIDI0LjkgNS4xIDMzLjkgMTQuMWw2Ny45IDY3LjljOSA5IDE0LjEgMjEuMiAxNC4xIDMzLjlMNDQ4IDMzNmMwIDI2LjUtMjEuNSA0OC00OCA0OGwtMTkyIDBjLTI2LjUgMC00OC0yMS41LTQ4LTQ4bDAtMjg4YzAtMjYuNSAyMS41LTQ4IDQ4LTQ4ek00OCAxMjhsODAgMCAwIDY0LTY0IDAgMCAyNTYgMTkyIDAgMC0zMiA2NCAwIDAgNDhjMCAyNi41LTIxLjUgNDgtNDggNDhMNDggNTEyYy0yNi41IDAtNDgtMjEuNS00OC00OEwwIDE3NmMwLTI2LjUgMjEuNS00OCA0OC00OHoiLz48L3N2Zz4=);
    background-repeat: no-repeat;
    background-size: 14px 14px;
    background-position: 0;
    display: inline-block;
    width: 14px;
    height: 14px;
    cursor: pointer;
}
</style><body><div id="sk-container-id-2" class="sk-top-container"><div class="sk-text-repr-fallback"><pre>LogisticRegression(max_iter=300)</pre><b>In a Jupyter environment, please rerun this cell to show the HTML representation or trust the notebook. <br />On GitHub, the HTML representation is unable to render, please try loading this page with nbviewer.org.</b></div><div class="sk-container" hidden><div class="sk-item"><div class="sk-estimator fitted sk-toggleable"><input class="sk-toggleable__control sk-hidden--visually" id="sk-estimator-id-2" type="checkbox" checked><label for="sk-estimator-id-2" class="sk-toggleable__label fitted sk-toggleable__label-arrow"><div><div>LogisticRegression</div></div><div><a class="sk-estimator-doc-link fitted" rel="noreferrer" target="_blank" href="https://scikit-learn.org/1.7/modules/generated/sklearn.linear_model.LogisticRegression.html">?<span>Documentation for LogisticRegression</span></a><span class="sk-estimator-doc-link fitted">i<span>Fitted</span></span></div></label><div class="sk-toggleable__content fitted" data-param-prefix="">
        <div class="estimator-table">
            <details>
                <summary>Parameters</summary>
                &#10;

|     |                    |              |
|-----|--------------------|--------------|
|     | penalty            | 'l2'         |
|     | dual               | False        |
|     | tol                | 0.0001       |
|     | C                  | 1.0          |
|     | fit_intercept      | True         |
|     | intercept_scaling  | 1            |
|     | class_weight       | None         |
|     | random_state       | None         |
|     | solver             | 'lbfgs'      |
|     | max_iter           | 300          |
|     | multi_class        | 'deprecated' |
|     | verbose            | 0            |
|     | warm_start         | False        |
|     | n_jobs             | None         |
|     | l1_ratio           | None         |

            </details>
        </div>
    </div></div></div></div></div><script>function copyToClipboard(text, element) {
    // Get the parameter prefix from the closest toggleable content
    const toggleableContent = element.closest('.sk-toggleable__content');
    const paramPrefix = toggleableContent ? toggleableContent.dataset.paramPrefix : '';
    const fullParamName = paramPrefix ? `${paramPrefix}${text}` : text;
&#10;    const originalStyle = element.style;
    const computedStyle = window.getComputedStyle(element);
    const originalWidth = computedStyle.width;
    const originalHTML = element.innerHTML.replace('Copied!', '');
&#10;    navigator.clipboard.writeText(fullParamName)
        .then(() => {
            element.style.width = originalWidth;
            element.style.color = 'green';
            element.innerHTML = "Copied!";
&#10;            setTimeout(() => {
                element.innerHTML = originalHTML;
                element.style = originalStyle;
            }, 2000);
        })
        .catch(err => {
            console.error('Failed to copy:', err);
            element.style.color = 'red';
            element.innerHTML = "Failed!";
            setTimeout(() => {
                element.innerHTML = originalHTML;
                element.style = originalStyle;
            }, 2000);
        });
    return false;
}
&#10;document.querySelectorAll('.fa-regular.fa-copy').forEach(function(element) {
    const toggleableContent = element.closest('.sk-toggleable__content');
    const paramPrefix = toggleableContent ? toggleableContent.dataset.paramPrefix : '';
    const paramName = element.parentElement.nextElementSibling.textContent.trim();
    const fullParamName = paramPrefix ? `${paramPrefix}${paramName}` : paramName;
&#10;    element.setAttribute('title', fullParamName);
});
</script></body>

Next I evaluated the performance of the model. I got an acuracy of .67,
but also a precision score of 0.00 for negative sentiments becasue it
labeled all negative sentences incorrectly. This is probably due to the
training set being so small.

``` python
y_pred = clf.predict(X_test_vec)

print("Accuracy:", accuracy_score(y_test, y_pred))
print(classification_report(y_test, y_pred))
```

    Accuracy: 0.6666666666666666
                  precision    recall  f1-score   support

        negative       0.00      0.00      0.00         4
        positive       0.67      1.00      0.80         8

        accuracy                           0.67        12
       macro avg       0.33      0.50      0.40        12
    weighted avg       0.44      0.67      0.53        12

    C:\Users\jilli\AppData\Local\Programs\Python\Python311\Lib\site-packages\sklearn\metrics\_classification.py:1731: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
      _warn_prf(average, modifier, f"{metric.capitalize()} is", result.shape[0])
    C:\Users\jilli\AppData\Local\Programs\Python\Python311\Lib\site-packages\sklearn\metrics\_classification.py:1731: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
      _warn_prf(average, modifier, f"{metric.capitalize()} is", result.shape[0])
    C:\Users\jilli\AppData\Local\Programs\Python\Python311\Lib\site-packages\sklearn\metrics\_classification.py:1731: UndefinedMetricWarning: Precision is ill-defined and being set to 0.0 in labels with no predicted samples. Use `zero_division` parameter to control this behavior.
      _warn_prf(average, modifier, f"{metric.capitalize()} is", result.shape[0])

This is the last step where I used the model on each country’s wikipedia
page. I turned each whole text into a TF-IDF vector using the same
vocabulary. Then I gave probabilities to each class of pos or neg. Next
I extracted those probabilities to create the sentiment analysis score
and finally gave a hard label to the sentiment score. The logistic
regression model results point to all pages having a positive sentiment
score. The highest growth country did have the highest sentiment score
which was a positive. However there was no clear distinction of
sentiment among high growth and low growth groups.

``` python
wiki_vec = tfidf.transform(wiki_df["Text"])
probs = clf.predict_proba(wiki_vec)

wiki_df["ML_Sentiment_Score"] = probs[:, list(clf.classes_).index("positive")]
wiki_df["ML_Sentiment_Label"] = clf.predict(wiki_vec)

wiki_df[["Country", "ML_Sentiment_Score", "ML_Sentiment_Label"]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | ML_Sentiment_Score | ML_Sentiment_Label |
|-----|---------------|--------------------|--------------------|
| 0   | Philippines   | 0.801810           | positive           |
| 1   | Mexico        | 0.699601           | positive           |
| 2   | United States | 0.771709           | positive           |
| 3   | Brazil        | 0.695276           | positive           |
| 4   | Poland        | 0.681315           | positive           |
| 5   | Spain         | 0.775712           | positive           |
| 6   | Italy         | 0.726184           | positive           |
| 7   | France        | 0.689739           | positive           |

</div>

# Regex content extraction

I wanted to keep trying to see if I could find more details that would
help me solve the question. This time I used regex mthods to get the
dates during the time period of 1910-2010.

``` python
import re

def extract_events_1910_2010(text):
    if not isinstance(text, str):
        return []

    year_regex = r"(19[1-9]\d|200\d|2010)"
    pattern = year_regex + r".{0,80}"

    matches = re.findall(pattern, text)

    return matches
```

Only 4 countries had dates show up.

``` python
wiki_df["Events_1910_2010"] = wiki_df["Text"].apply(extract_events_1910_2010)
wiki_df[["Country", "Events_1910_2010"]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Events_1910_2010     |
|-----|---------------|----------------------|
| 0   | Philippines   | \[\]                 |
| 1   | Mexico        | \[\]                 |
| 2   | United States | \[\]                 |
| 3   | Brazil        | \[2010, 1991\]       |
| 4   | Poland        | \[2000\]             |
| 5   | Spain         | \[1978, 1976, 1953\] |
| 6   | Italy         | \[1985\]             |
| 7   | France        | \[\]                 |

</div>

Then I picked out the sentences surrounding those years to see what
further details might appear. The results here aren’t super insightful.
The only one that really sticks out to me is how in the year 2000 Poland
had baptized 99% of all children born.

``` python
def extract_events_1910_2010_structured(text):
    if not isinstance(text, str):
        return []
    
    pattern = r"(19[1-9]\d|200\d|2010)(.{0,80})"
    results = []

    for match in re.findall(pattern, text):
        year = match[0]
        context = match[1].strip()
        results.append((int(year), context))

    return results

wiki_df["Events_1910_2010_structured"] = wiki_df["Text"].apply(extract_events_1910_2010_structured)
wiki_df[["Country", "Events_1910_2010_structured"]]
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }
&#10;    .dataframe tbody tr th {
        vertical-align: top;
    }
&#10;    .dataframe thead th {
        text-align: right;
    }
</style>

|     | Country       | Events_1910_2010_structured                        |
|-----|---------------|----------------------------------------------------|
| 0   | Philippines   | \[\]                                               |
| 1   | Mexico        | \[\]                                               |
| 2   | United States | \[\]                                               |
| 3   | Brazil        | \[(2010, , when 65.1% of the population aged 10... |
| 4   | Poland        | \[(2000, s, 99% of all children born in Poland ... |
| 5   | Spain         | \[(1978, establishes the non-denominationality ... |
| 6   | Italy         | \[(1985, , with the renegotiation of the Latera... |
| 7   | France        | \[\]                                               |

</div>

## Results

A summary of key findings and visualizations is included in the rendered
output of `final_project.qmd`.

Conclusion: My original question was trying to understand why different
country’s Catholic populations grew exponentially while others
diminished. I went through a long series of Unstructured Analytic
methods in order to find the answer.

First I started by scraping the Pew research center for 2 images that
detailed statistics from 1910 to 2010. From there I used Optical
Character Recognition to try and scrape the information from the images.
I would have just scraped the words from the article itself, however the
data was only contained in the images. Which led to a fun unstructured
problem of downloading and using the Tesseract, which made me feel like
I was in a Marvel Avengers movie. OCR made a few mistakes which I
corrected and then I computed the growth factor and percent change. The
Philippines had the greatest then following in order was Mexico, the US,
Brazil, Poland, Spain, Italy, and France. It was quite interesting
because France had the largest Catholic population in 1910 but then
“fell from the graces” of country population growth lol.

From there I scraped the Wikipedia pages for each country concerning the
presence of the Catholic Church. I did multiple types of Topic modeling,
sentiment analyses, and regex content extraction. My hope was that there
would be a glaringly clear conclusion. There never really was which was
why I was so motivated to try out so many variations of the unstructured
analytics tools that we learned in class. The most insightful tool was
topic modeling attempt \#3. High growth countries fell into topic 0, low
growth countries fell into topic 1. Topic 0 had themes of
evangelization, while topic 1 had themes of historical influence. Which
would correlate with findings of growth vs decay.

I had been hoping that the sentiment analysis might find overwhelmingly
negative and positive words with a greater divide. Unfortunately
Wikipedia pages contain pretty neutral text so most of the analysis was
pretty balanced among high and low growth. The last method of regex
content extraction was used in hopes that there might be a stark
historical event that may have had a large impact on these population
numbers. Unfortunately that wasn’t much of the case either.

All in all, this project was a test of patience and perseverance with
little to show for it. I wish I had better results, but it was a fun
process exploring ideas from class in the context of global Catholic
history.

## Reproducibility

All code and data necessary to reproduce the analysis are contained in
this repository. Thank you!
