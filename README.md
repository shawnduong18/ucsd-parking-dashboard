# Parking Occupancy Dashboard for UC San Diego

One of the most common frustrations for myself during my time at San Diego was driving and finding parking.

This project started as a data visualization challenge and transformed into a full interactive web dashboard with a machine learning model behind it. By the end, it became a tool that lets any UCSD commuter enter their destination, arrival time, and permit type, and get a ranked prediction of which nearby lots are most likely to have open spaces.

---

## Data Sources and Tools

I used the following data sources:

**UCSD Transportation Services** provided 20 quarterly parking occupancy survey files spanning Fall 2019 through Winter 2025–26. Each file contains occupancy records at hourly intervals throughout a single survey day, broken down by lot and permit type.

I used the following tools for my analysis and modeling:

**Python** was the backbone of the entire data pipeline. I used it to clean and combine the source files, engineer features for the machine learning model, and export pre-computed predictions for the dashboard.

**pandas + openpyxl** handled all Excel file reading, schema normalization, and output formatting. With 20 source files across three distinct format generations, these libraries did most of the heavy lifting.

**scikit-learn** was used to train and evaluate the classification model specifically `GradientBoostingRegressor` for the prediction table and `RandomForestClassifier`, `LogisticRegression`, and `KNeighborsClassifier` for the model comparison analysis.

**Matplotlib + Seaborn** were used to generate exploratory visualizations including heatmaps, box plots, and trend lines from the cleaned dataset.

**Chart.js** powers all of the interactive charts in the web dashboard.

**Leaflet** renders the interactive geographic map. It is a lightweight (~40 KB), open-source mapping library that runs entirely client-side with no API key required.
 
**Overpass API** is queried directly from the browser at page load to fetch the current set of UCSD parking lots and structures from OpenStreetMap. Results are cached in `localStorage` for 7 days so subsequent visits load instantly without re-querying the public OSM servers.

**HTML, CSS, and JavaScript** make up the entire front end. The dashboard is a single self-contained file with no server, no backend, and no external API dependencies, making it deployable for free on GitHub Pages.

---

## Data Cleaning

Before any analysis could happen, the data had to be cleaned which turned out to be the most challenging part of this project. 

Initially I wanted to use parking data dating all the way back to 2000, however the sheer amount of data as well as the varying file formats from legacy records made the data consoloidation extremely difficult. I opted to reduce the scope of the data to 2019 to 2026 to make the process easier as well as keep parking trends consistent. 

The 20 source files came in **three distinct format generations**. The older `.xlsx` files from 2019–2022 used one sheet layout. A transitional format appeared in 2022–2023 with pivot table structures and seasonally named sheets. The newer `.xlsm` macro-enabled files from 2024 onward were redesigned entirely, with a raw `Pivot Table Data` sheet replacing the aggregated summaries.

Each generation required its own parsing logic. A single unified script would try to open each file, inspect the sheet names, and route it to the correct parser.

Beyond the format differences, the data had other issues: survey footnotes being parsed as lot IDs, space type labels like `"D ($5.00)"` that needed to be normalized to `"D"`, and numeric lot IDs from the SIO area that looked like data errors but were actually legitimate lot numbers.

After cleaning, the combined dataset produced **289 rows** at the university-wide level and **13,033 rows** at the lot level one row per lot, per permit type, per survey period.

It's also worth noting: the **2023–24 academic year is completely absent** from the source files. There is a one-year gap between Spring 2022–23 and Summer 2024–25 that appears in every chart.

---

## Exploratory Visualizations

With the data cleaned, I created visualizations to better understand parking behavior at UC San Diego. 

University-wide Peak Occupancy by Quarter is the main visualization, a line chart plotting peak-hour occupancy (as a percentage) across 20 academic quarters from Fall 2019 through Winter 2025–26. Each data point is color-coded by season: gold for Fall, navy for Winter, green for Spring, and red triangles for Summer. A key insight callout above the chart highlights the pandemic's impact: occupancy collapsed from 91.1% in Winter 2019–20 to 40.2% by Spring 2020, and recovery has been steady but incomplete, with Winter 2025–26 sitting 26 percentage points below the pre-pandemic peak. A gap annotation flags the missing 2023–24 academic year from the source data.

Average by Season is a four-bar chart showing mean peak occupancy grouped by academic quarter across all available years. The bars are color-matched to the season scheme above, making it easy to see at a glance which quarters consistently see the heaviest demand.

Surface Lots vs. Structures is a dual-line chart overlaying the same quarterly timeline but splitting it into two series: surface lots (blue) and parking structures (green). The subtitle notes the key finding directly, structures run 5–10 percentage points higher than surface lots across the dataset, reflecting that structures serve more central, high-demand areas of campus.

Peak Occupancy by Campus Area is a horizontal bar chart ranking ten campus zones by their typical weekday peak demand, averaged over post-pandemic quarters. Bars are color-coded by demand tier: red (≥80%) for the highest-pressure areas like Geisel/Price Center (87.5%) and Warren/Sixth (83.8%), amber for mid-range zones, and green for lower-demand areas like SIO/Scripps (54.8%). This makes the geographic distribution of parking pressure immediately legible.

---

## Implementing Machine Learning

The model frames the prediction as a **binary classification problem**: given a lot, permit type, arrival hour, and academic quarter will a spot be available? A spot is defined as available when occupancy is below 90%.

Three models were tested and compared using 5-fold stratified cross-validation:

| Model | Accuracy | Precision | Recall | F1 Score |
|---|---|---|---|---|
| Logistic Regression | 71.8% | 71.8% | 100.0% | 83.6% |
| Random Forest | 73.6 | 74.4% | 96.6% | 84.0% |
| K-Nearest Neighbors | 72.2% | 75.8% | 90.1% | 82.3% |
| **XGBoost** | **73.1%** | **76.4%** | **86.8%** | **81.3%** |

**XGBoost** was selected as the final model. It achieved the highest accuracy of all four algorithms, a 2.3-point improvement over Random Forest and 5.8 points over Logistic Regression. More importantly, the accuracy gain came from better precision. When XGBoost says a lot is available, it's right 76.4% of the time, compared to 71.4% for Random Forest.

Feature importance analysis from the Random Forest revealed that **hour of day** and **campus location** are the two strongest predictors, followed by academic quarter and permit type. This makes intuitive sense: the time of day drives the core occupancy curve, and different parts of campus have very different baseline demand.

Once the ideal model was selected, it was used to generate a **pre-computed prediction table**: one predicted occupancy probability for every combination of lot, permit type, hour, and season. That table containing roughly 3,240 entries is what gets embedded directly into the dashboard. The model itself never runs in the browser.

---

## The Dashboard

The final product is a single HTML file deployable on GitHub Pages. https://shawnduong18.github.io/ucsd-parking-dashboard/ It opens on a **Find Parking** tab showing an interactive stylized campus map, with lot pins color-coded from green (likely open) to red (usually full) based on the ML model's predictions.

A user selects their destination from a dropdown, sets their arrival time with a slider, picks their permit type, and hits Find Parking. The map highlights nearby lots, dims the rest, and produces a ranked list of options below. Clicking any pin shows a popup with the full hourly occupancy profile for that lot.

An **Overview** tab dispalying the university-wide occupancy trend across all 20 survey periods.

An **About** tab documents the data source, permit type definitions, and known limitations for users unfamiliar with the survey methodology.

---

## Challenges and Limitations

**One survey day per quarter** is the most significant constraint. The model learns historical patterns, not real-time conditions. Events, construction, weather, and exam periods are not taking into consideration. The accuracy ceiling across all three models sits between 72–74%.

**Format drift across six years** required significant reverse-engineering. Each format generation made different assumptions about how to organize lot-level data, and several fields changed names, positions, or granularity between versions. Additionally, some parking lots were not included in the data however they are present on the dashboard through the OpenStreetMaps API so they aren't included in predictions.

**Lot naming inconsistencies** meant the same physical structure could appear as `(P341-7)` in a 2020 file and `Hopkins` in a 2024 file. A manual mapping was required to reconcile these.

**Structure-level data for 2024–2026** was not reliably extractable from the newer `.xlsm` files, which is why the Structures tab currently covers 2019–2023 only.

---

## Results

For a tool built entirely from one survey day per quarter, the results held up well. The model correctly identifies available spots about three quarters of the time, and when it predicts availability it has high recall meaning it rarely misses a genuinely open lot.

The campus map brings the data to life in a way that a table or chart cannot. Watching the pin colors shift from green to red as the time slider crosses 10am in Fall makes the occupancy pattern instantly tangible for users. 

---

## Conclusions and Next Steps

The purpose of this project goes further than to build a tool and gain insight, it was an oppertunity to contribute value to my community as well as refine my technical skills. 

The current iteration of the project is a foundation that can grow considerably with more data. Some directions worth exploring:

**Real-time sensor integration** would be the single most impactful improvement. Connecting to live occupancy feeds would transform the tool from a historical pattern guide into a day-of planning resource, something closer to the live traffic layer in Google Maps.

**Exam period and event flagging** would add an important layer of context. Parking during finals week behaves very differently from a typical Tuesday in October, and the current model has no way to distinguish between the two.

The full source code: three Python scripts and the HTML dashboard is available on GitHub. Running the pipeline on updated survey files automatically retrains the model and regenerates the prediction table, so keeping the dashboard current requires nothing more than dropping in the new file and running the scripts.

---

*Data source: UCSD Transportation Services Occupancy Survey Files, 2019–2026.*
