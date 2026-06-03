# Building a Parking Occupancy Dashboard for UC San Diego

One of the most common frustrations for students and staff at UC San Diego is finding parking. Anyone who has circled a lot for twenty minutes before an 8am class knows the feeling. The question I kept coming back to was: *is there any pattern to this?* And if there is — could that pattern be turned into something useful?

This project started as a data cleaning challenge and grew into a full interactive web dashboard with a machine learning model underneath it. By the end, it became a tool that lets any UCSD commuter enter their destination, arrival time, and permit type, and get a ranked prediction of which nearby lots are most likely to have open spaces.

---

## Data Sources and Tools

I used the following data sources:

**UCSD Transportation Services** provided 20 quarterly parking occupancy survey files spanning Fall 2019 through Winter 2025–26. Each file records physical stall counts — occupied and empty — at 10 hourly snapshots throughout a single survey day, broken down by lot and permit type.

And I used the following tools:

**Python** was the backbone of the entire data pipeline. I used it to clean and combine the source files, engineer features for the machine learning model, and export pre-computed predictions for the dashboard.

**pandas + openpyxl** handled all Excel file reading, schema normalization, and output formatting. With 20 source files across three distinct format generations, these libraries did most of the heavy lifting.

**scikit-learn** was used to train and evaluate the classification model — specifically `GradientBoostingRegressor` for the prediction table and `RandomForestClassifier`, `LogisticRegression`, and `KNeighborsClassifier` for the model comparison analysis.

**Matplotlib + Seaborn** were used to generate exploratory visualizations including violin plots, heatmaps, box plots, and trend lines from the cleaned dataset.

**Chart.js** powers all of the interactive charts in the web dashboard — line charts, bar charts, and horizontal rankings.

**HTML, CSS, and JavaScript** make up the entire front end. The dashboard is a single self-contained file with no server, no backend, and no external API dependencies, making it deployable for free on GitHub Pages.

---

## The Data Challenge

Before any analysis could happen, the data had to be cleaned — and that turned out to be the most significant engineering challenge of the project.

The 20 source files came in **three distinct format generations**. The older `.xlsx` files from 2019–2022 used one sheet layout. A transitional format appeared in 2022–2023 with pivot table structures and seasonally named sheets. The newer `.xlsm` macro-enabled files from 2024 onward were redesigned entirely, with a raw `Pivot Table Data` sheet replacing the aggregated summaries.

Each generation required its own parsing logic. A single unified script would try to open each file, inspect the sheet names, and route it to the correct parser — a pattern that ended up being one of the more satisfying parts of the build.

Beyond the format differences, the data had other issues: survey footnotes being parsed as lot IDs, space type labels like `"D ($5.00)"` that needed to be normalized to `"D"`, and numeric lot IDs from the SIO area that looked like data errors but were actually legitimate lot numbers.

After cleaning, the combined dataset produced **289 rows** at the university-wide level and **13,033 rows** at the lot level — one row per lot, per permit type, per survey period.

It's also worth noting: the **2023–24 academic year is completely absent** from the source files. There is a one-year gap between Spring 2022–23 and Summer 2024–25 that appears in every chart.

---

## Exploratory Visualizations

With the data cleaned, the first step was understanding what it actually showed.

The most striking finding was visible in a single trend line: university-wide peak occupancy hit **91.1% in Winter 2019–20**, then collapsed to **40.2% in Spring 2019–20** when COVID emptied the campus. It has never fully recovered — recent periods plateau around 63–67%.

A seasonal heatmap (hour × quarter) made the daily occupancy pattern immediately clear. The **10am–12pm window in Fall and Winter** is consistently the darkest block — this became one of the key features in the machine learning model.

A box plot of peak occupancy by permit type revealed something interesting about SR (Special Reserved) spaces: they are nearly 100% occupied in almost every period, with almost no spread. Reserved permit spaces, on the other hand, hover around 35–50% with high variability — suggesting chronic oversupply.

A correlation matrix of the hourly stall counts confirmed strong positive correlations across adjacent hours (0.95+), validating the assumption that occupancy builds and drops smoothly throughout the day rather than spiking randomly.

---

## The Machine Learning Model

The model frames the prediction as a **binary classification problem**: given a lot, permit type, arrival hour, and academic quarter — will a spot be available? A spot is defined as available when occupancy is below 90%.

Three models were tested and compared using 5-fold stratified cross-validation:

| Model | Accuracy | F1 Score |
|---|---|---|
| Logistic Regression | 71.8% | 83.6% |
| **Random Forest** | **73.6%** | **84.0%** |
| K-Nearest Neighbors | 72.2% | 82.3% |

**Random Forest** was selected as the final model based on its F1 score, which balances precision and recall. For a parking tool, recall matters more than precision — it's worse to tell a student a lot is full when it has spaces than to occasionally send them somewhere that turns out to be fuller than expected.

Feature importance analysis from the Random Forest revealed that **hour of day** and **campus location** are the two strongest predictors, followed by academic quarter and permit type. This makes intuitive sense: the time of day drives the core occupancy curve, and different parts of campus have very different baseline demand.

Once the winning model was selected, it was used to generate a **pre-computed prediction table**: one predicted occupancy probability for every combination of lot, permit type, hour, and season. That table — roughly 3,240 entries — is what gets embedded directly into the dashboard. The model itself never runs in the browser.

---

## The Dashboard

The final product is a single HTML file deployable on GitHub Pages. It opens on a **Find Parking** tab showing an interactive stylized campus map, with lot pins color-coded from green (likely open) to red (usually full) based on the ML model's predictions.

A user selects their destination from a dropdown, sets their arrival time with a slider, picks their permit type, and hits Find Parking. The map highlights nearby lots, dims the rest, and produces a ranked list of options below. Clicking any pin shows a popup with the full hourly occupancy profile for that lot.

The rest of the dashboard is organized into a **Trends** dropdown with three sub-pages:

- **Overview** — the university-wide occupancy trend across all 20 survey periods, with the COVID drop and recovery arc clearly visible
- **By Space Type** — a daily profile chart comparing how different permit types fill up hour by hour in any selected period
- **Structures** — hourly occupancy curves for named parking structures like Gilman, Hopkins, Pangea, and South, covering 2019–2023

An **About** tab documents the data source, permit type definitions, and known limitations for users unfamiliar with the survey methodology.

---

## Challenges and Limitations

**One survey day per quarter** is the most significant constraint. The model learns historical patterns, not real-time conditions. Events, construction, weather, and exam periods are invisible to it. The accuracy ceiling across all three models sits between 72–74%, and that gap is a data limitation — not an algorithmic one.

**Format drift across six years** required significant reverse-engineering. Each format generation made different assumptions about how to organize lot-level data, and several fields changed names, positions, or granularity between versions.

**Lot naming inconsistencies** meant the same physical structure could appear as `(P341-7)` in a 2020 file and `Hopkins` in a 2024 file. A manual mapping was required to reconcile these.

**Structure-level data for 2024–2026** was not reliably extractable from the newer `.xlsm` files, which is why the Structures tab currently covers 2019–2023 only.

---

## Results

For a tool built entirely from one survey day per quarter, the results held up well. The model correctly identifies available spots about three quarters of the time, and when it predicts availability it has high recall — it rarely misses a genuinely open lot.

The campus map brings the data to life in a way that a table or chart cannot. Watching the pin colors shift from green to red as the time slider crosses 10am in Fall makes the occupancy pattern instantly tangible.

---

## Conclusions and Next Steps

This project laid a foundation that can grow considerably with more data. Some directions worth exploring:

**Real-time sensor integration** would be the single most impactful improvement. Connecting to live occupancy feeds would transform the tool from a historical pattern guide into a day-of planning resource — something closer to the live traffic layer in Google Maps.

**More survey days per quarter** would meaningfully reduce the model's uncertainty. Even two or three survey days would allow the model to learn day-of-week effects and separate typical days from unusual ones.

**A Google Maps or campus schematic overlay** for the interactive map would replace the current stylized campus drawing with accurate lot boundaries and real geography, making the Find Parking tool significantly more intuitive.

**Exam period and event flagging** would add an important layer of context. Parking during finals week behaves very differently from a typical Tuesday in October, and the current model has no way to distinguish between the two.

The full source code — three Python scripts and the HTML dashboard — is available on GitHub. Running the pipeline on updated survey files automatically retrains the model and regenerates the prediction table, so keeping the dashboard current requires nothing more than dropping in the new file and running the scripts.

---

*Data source: UCSD Transportation Services Occupancy Survey Files, 2019–2026. Built with Python, scikit-learn, Chart.js, and a lot of Excel.*
