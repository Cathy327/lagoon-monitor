# User Guide — Lagoon Water Quality Monitor

## What This Tool Does

This dashboard displays satellite-derived water quality indicators for the lagoon, using imagery from the European Space Agency's Sentinel-2 satellites. It shows two indices:

- **FAI (Floating Algae Index)**: Highlights potential floating algae on the water surface. Useful for spotting short-term bloom events.
- **NDCI (Normalised Difference Chlorophyll Index)**: Estimates chlorophyll concentration as a measure of algal biomass.

New satellite images become available approximately every 5 days, weather permitting. During periods of high cloud cover (common in winter), there may be gaps of several weeks.

---

## Viewing the Dashboard

1. Open the `index.html` file in your web browser (Chrome, Safari, Firefox, or Edge).
2. The dashboard loads automatically with all available data.

### Satellite Imagery (Top Section)

- Use the **dropdown menu** to select a date, or click the **‹ ›** arrows to step through available dates.
- The left map shows **FAI**, the right shows **NDCI** for the same date.
- Colour bars below each map show what the colours represent (blue = low values, red = high values).

### Three-Year Comparison (Middle Section)

- Select an end date using the dropdown. The dashboard shows all available images from one month before that date, grouped by year (2024, 2025, 2026).
- Click any thumbnail image to view it enlarged.
- This allows you to compare conditions during the same period across different years.

### Trend Charts (Below Comparison)

- Two line charts show the full history of FAI and NDCI values from 2024 to present.
- The **solid line** is the average value across the lagoon for each date.
- The **shaded band** shows the standard deviation (how much variation exists within the lagoon).
- **Coloured highlights** on the chart correspond to the comparison window selected above.
- **Dashed vertical lines** mark any bloom events you have recorded (see below).

### Bloom Event Log (Bottom Section)

- Click **Record event** to log an observed algal bloom.
- Enter the date, severity (Low / Medium / High), and any notes about your observations.
- Recorded events appear as markers on the trend charts.
- Events are saved in your browser and persist between sessions.

---

## Updating the Data

When you want to add the latest satellite imagery to the dashboard:

### Step 1: Open the Update Tool

- Go to the project page on GitHub (link provided by the project team).
- Click the **Open in Colab** button.
- Log in with the Google account provided by the project team when prompted.

### Step 2: Run the Update

- In Google Colab, click the menu: **Runtime → Run all**.
- You will see a popup asking to authorise access to Google Drive — click **Allow**.
- The tool will automatically:
  - Check what data you already have (stored on Google Drive).
  - Download any new satellite imagery that has become available.
  - Generate updated charts and metadata.
- Progress is displayed in the notebook. A typical incremental update takes 2–5 minutes.

### Step 3: Download the Updated Data

- When processing is complete, a zip file will automatically download.
- If it does not download automatically, look for a download link at the bottom of the notebook.

### Step 4: Update Your Dashboard

1. Find the downloaded zip file (usually in your Downloads folder). It will be named `lagoon_dashboard_data.zip`.
2. Extract (unzip) the file. You will get a folder called `data`.
3. Go to the folder where your dashboard is stored (where `index.html` is located).
4. **Delete** the existing `data` folder.
5. **Move** the new `data` folder into the dashboard folder, replacing the old one.
6. Open `index.html` in your browser (or refresh if it is already open).

You should now see the latest satellite imagery and updated charts.

---

## Troubleshooting

### "No new images available"
This is normal during cloudy periods. Sentinel-2 images with more than 30% cloud cover are filtered out. This is common during winter months (June–August). The dashboard will show the date of the most recent usable image.

### The dashboard shows a blank page
Make sure the `data` folder is in the same directory as `index.html` and contains the `dashboard_data.js` file.

### Maps do not display background tiles
The map background requires an internet connection to load. The satellite imagery overlays will still display without internet.

### Colab asks to log in again
Google authentication tokens expire periodically. Simply log in again with the provided account when prompted.

### Bloom events are not appearing on charts
Bloom event markers only appear within the time range of the chart. If you recorded an event for a date outside the chart's range, it will not be visible.

---

## Contact

For technical issues with the dashboard or data pipeline, contact the project team at the University of Melbourne.
