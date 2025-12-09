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
