# 🎥 Top 10 UK YouTubers 2025  
*Helping marketing managers identify the best UK channels to partner with.*

---

## 📚 Table of Contents
1. [Overview](#overview)
2. [Raw Data](#raw-data)
3. [Data Collection (Python Script)](#data-collection-python-script)
4. [New Data (Processed)](#new-data-processed)
5. [Interactive Dashboard](#interactive-dashboard)
6. [Tools Used](#tools-used)
7. [Insights & Recommendations](#insights--recommendations)
8. [How to Reproduce](#how-to-reproduce)
9. [Contact](#contact)

---

## 🧩 Overview
This project analyzes the top 10 UK YouTubers in 2025 to help marketing managers make data-driven partnership decisions.  
The dataset includes subscriber counts, average views, video engagement metrics, and content categories.

---

## 📊 Raw Data
The initial dataset was compiled from publicly available YouTube API data.

📂 **[Download Raw Data (CSV)](Assets/Datasets/youtube top channels from kaggle.csv)**



---

## 🐍 Data Collection (Python Script)

```python
from googleapiclient.discovery import build
import pandas as pd

# --- CONFIG ---
API_KEY = "I454323245tv477O2KRu6QCJnAM"
Excel_file = r"C:\Users\USER\Desktop\youtube top channels.xlsx"

# --- Load Excel ---
df_channels = pd.read_excel(Excel_file)
df_channels.columns = df_channels.columns.str.strip()  # clean header spaces

# make sure the columns exist
if "Channel Name" not in df_channels.columns or "Channel ID" not in df_channels.columns:
    raise ValueError("❌ Excel must have columns: 'Channel Name' and 'Channel ID'")

# Build YouTube API
youtube = build("youtube", "v3", developerKey=API_KEY)

# --- Step 1: Fill missing Channel IDs ---
for i, row in df_channels.iterrows():
    ch_id = str(row["Channel ID"]).strip() if not pd.isna(row["Channel ID"]) else ""
    if ch_id == "" or ch_id.lower() in ["not found", "error"]:
        ch_name = str(row["Channel Name"]).strip()
        print(f"🔍 Searching Channel ID for: {ch_name}")
        try:
            request = youtube.search().list(
                q=ch_name,
                part="snippet",
                type="channel",
                maxResults=1
            )
            response = request.execute()

            if response.get("items"):
                channel_id = response["items"][0]["snippet"]["channelId"]
                df_channels.at[i, "Channel ID"] = channel_id
                print(f"✅ Found ID: {channel_id}")
            else:
                df_channels.at[i, "Channel ID"] = "Not Found"
                print(f"❌ Not found: {ch_name}")
        except Exception as e:
            df_channels.at[i, "Channel ID"] = "Error"
            print(f"⚠️ Error fetching ID for {ch_name}: {e}")

# --- Step 2: Fetch stats for all valid Channel IDs ---
def get_channel_stats(youtube, channel_ids):
    all_data = []
    for i in range(0, len(channel_ids), 50):
        batch = channel_ids[i:i+50]
        request = youtube.channels().list(
            part="snippet,statistics",
            id=",".join(batch)
        )
        response = request.execute()

        for item in response.get("items", []):
            data = {
                "Channel ID": item["id"],
                "Channel Name (API)": item["snippet"]["title"],
                "Subscribers": int(item["statistics"].get("subscriberCount", 0)),
                "Total Views": int(item["statistics"].get("viewCount", 0)),
                "Total Videos": int(item["statistics"].get("videoCount", 0))
            }
            all_data.append(data)
    return all_data

# clean and filter valid IDs
valid_ids = df_channels["Channel ID"]
valid_ids = valid_ids[~valid_ids.isin(["Not Found", "Error"])]
valid_ids = valid_ids[valid_ids.notna()].astype(str).str.strip().tolist()

print(f"\n📊 Fetching stats for {len(valid_ids)} valid channels...\n")

channel_data = get_channel_stats(youtube, valid_ids)
df_stats = pd.DataFrame(channel_data)

# --- Step 3: Merge results safely ---
df_final = pd.merge(df_channels, df_stats, on="Channel ID", how="left")

# --- Step 4: Overwrite same Excel file ---
with pd.ExcelWriter(Excel_file, engine="openpyxl", mode="w") as writer:
    df_final.to_excel(writer, index=False)

print("\n✅ Excel file successfully updated with missing IDs and stats!")
print(f"📁 Saved at: {Excel_file}")
```



