```python
import requests
import pandas as pd
import matplotlib.pyplot as plt
```

# Data Preparation
Query  MongoDB to put data into a dataframe


```python
#converting api endpoints to data frames
owners_url = "https://www.fema.gov/api/open/v2/HousingAssistanceOwners"
renters_url = "https://www.fema.gov/api/open/v2/HousingAssistanceRenters"

def fetch_all(url, params, key):
    all_data = []
    skip = 0
    
    while True:
        params["$skip"] = skip
        response = requests.get(url, params=params).json()[key]
        
        if not response:
            break
            
        all_data.extend(response)
        
        if len(response) < 1000:
            break
            
        skip += 1000
    
    return all_data

owner_params = {
    "$filter": "disasterNumber eq 4827",
    "$select": "state,zipCode,validRegistrations,approvedForFemaAssistance,totalApprovedIhpAmount,totalInspected,totalDamage",
    "$top": 1000,
    "$format": "json"
}

renter_params = {
    "$filter": "disasterNumber eq 4827",
    "$select": "state,zipCode,validRegistrations,approvedForFemaAssistance,totalApprovedIhpAmount,totalInspected,totalInspectedWithNoDamage,totalWithModerateDamage,totalWithMajorDamage,totalWithSubstantialDamage",
    "$top": 1000,
    "$format": "json"
}

owners_df = pd.DataFrame(fetch_all(owners_url, owner_params.copy(), "HousingAssistanceOwners"))
renters_df = pd.DataFrame(fetch_all(renters_url, renter_params.copy(), "HousingAssistanceRenters"))

print(len(owners_df), len(renters_df))
```

    1409 1457



```python
import logging
import os

os.makedirs("logs", exist_ok=True)  #creates a logs directory if it doesn't exist
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("logs/pipeline.log"),  #writes to a file
        logging.StreamHandler()               #still prints to console
    ]
)

log = logging.getLogger('pipeline')
log.info("Pipeline initialized.")
```

    2026-04-28 20:30:48,018 - INFO - Pipeline initialized.



```python
#connect
from urllib.parse import quote_plus
import getpass
from pymongo import MongoClient
import certifi
try:
    user = quote_plus(getpass.getpass("MongoDB username: "))
    password = quote_plus(getpass.getpass("MongoDB password: "))
    #reusing connection string format from previous homework for simplicity
    connection_string = f"mongodb+srv://{user}:{password}@design-hw-9.ymzizmq.mongodb.net/?appName=design-hw-9"
    
    client = MongoClient(connection_string, tlsCAFile=certifi.where())
    log.info(f"Connected to MongoDB version: {client.server_info()['version']}")
except Exception as e:
    log.error(f"Failed to connect to MongoDB: {e}")
    raise

```

    2026-04-28 20:31:31,753 - INFO - Connected to MongoDB version: 8.0.21



```python
#insert data
try:
    db = client["fema_housing"]
    
    owners_collection = db["owners"]
    renters_collection = db["renters"]
    
    #clear existing data to avoid duplicates on re-run
    owners_collection.delete_many({})
    renters_collection.delete_many({})
    log.info("Cleared existing collections")
    
    owners_records = owners_df.to_dict(orient="records")
    renters_records = renters_df.to_dict(orient="records")
    
    owners_collection.insert_many(owners_records)
    log.info(f"Inserted {len(owners_records)} owner records")
    
    renters_collection.insert_many(renters_records)
    log.info(f"Inserted {len(renters_records)} renter records")

except Exception as e:
    log.error(f"Failed to insert data: {e}")
    raise

#verify
try:
    owner_count = owners_collection.count_documents({})
    renter_count = renters_collection.count_documents({})
    log.info(f"Verified - Owners: {owner_count}, Renters: {renter_count}")
except Exception as e:
    log.error(f"Verification failed: {e}")
    raise
```

    2026-04-28 20:31:42,010 - INFO - Cleared existing collections
    2026-04-28 20:31:48,010 - INFO - Inserted 1409 owner records
    2026-04-28 20:31:52,423 - INFO - Inserted 1457 renter records
    2026-04-28 20:31:52,564 - INFO - Verified - Owners: 1409, Renters: 1457



```python
doc = owners_collection.find_one()
print(doc)
```

    {'_id': ObjectId('69f1516fba161563c0093678'), 'state': 'NC', 'zipCode': '28701', 'validRegistrations': 2, 'approvedForFemaAssistance': 2, 'totalApprovedIhpAmount': 8220.63, 'totalInspected': 1, 'totalDamage': 575.63}


# Solution Analysis


```python
#compute different metrics for each zip code and compute a disparity score
import numpy as np
import scipy.stats as stats

def compute_metrics(owners_df, renters_df, min_registrations=5):
    renters_df = renters_df.copy()
    renters_df["weighted_damage"] = (
        pd.to_numeric(renters_df["totalWithModerateDamage"],    errors="coerce").fillna(0) * 1 +
        pd.to_numeric(renters_df["totalWithMajorDamage"],       errors="coerce").fillna(0) * 2 +
        pd.to_numeric(renters_df["totalWithSubstantialDamage"], errors="coerce").fillna(0) * 3
    )
 
    owners_zip = owners_df.groupby("zipCode").agg(
        owner_registrations=("validRegistrations",        "sum"),
        owner_approved=     ("approvedForFemaAssistance", "sum"),
        owner_amount=       ("totalApprovedIhpAmount",    "sum"),
        owner_damage=       ("totalDamage",                "sum"),
    ).reset_index()
 
    renters_zip = renters_df.groupby("zipCode").agg(
        renter_registrations=("validRegistrations",        "sum"),
        renter_approved=     ("approvedForFemaAssistance", "sum"),
        renter_amount=       ("totalApprovedIhpAmount",    "sum"),
        renter_damage=       ("weighted_damage",            "sum"),
    ).reset_index()
 
    merged = owners_zip.merge(renters_zip, on="zipCode")
    merged = merged[
        (merged["owner_registrations"]  >= min_registrations) &
        (merged["renter_registrations"] >= min_registrations)
    ].copy()
 
    merged["owner_approval_rate"]   = merged["owner_approved"]  / merged["owner_registrations"]
    merged["renter_approval_rate"]  = merged["renter_approved"] / merged["renter_registrations"]
    merged["owner_avg_award"]       = merged["owner_amount"]    / merged["owner_approved"].replace(0, np.nan)
    merged["renter_avg_award"]      = merged["renter_amount"]   / merged["renter_approved"].replace(0, np.nan)
    merged["owner_damage_per_reg"]  = merged["owner_damage"]    / merged["owner_registrations"]
    merged["renter_damage_per_reg"] = merged["renter_damage"]   / merged["renter_registrations"]
 
    merged["approval_gap"] = merged["owner_approval_rate"]  - merged["renter_approval_rate"]
    merged["award_gap"]    = merged["owner_avg_award"]       - merged["renter_avg_award"]
    merged["damage_gap"]   = merged["owner_damage_per_reg"]  - merged["renter_damage_per_reg"]
 
    merged["approval_gap_z"] = stats.zscore(merged["approval_gap"].fillna(0))
    merged["award_gap_z"]    = stats.zscore(merged["award_gap"].fillna(0))
    merged["damage_gap_z"]   = stats.zscore(merged["damage_gap"].fillna(0))
 
    merged["disparity_score"] = (
        merged["approval_gap_z"] +
        merged["award_gap_z"]    +
        merged["damage_gap_z"]
    ) / 3
 
    print(f"Computed metrics for {len(merged)} ZIP codes")
    return merged.reset_index(drop=True)
```


```python
merged = compute_metrics(owners_df, renters_df)
```

    Computed metrics for 269 ZIP codes



```python
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import seaborn as sns
import numpy as np
# define the three gap columns as model features and create a binary target for high vs low disparity
features = ["approval_gap", "award_gap", "damage_gap"]
X = merged[features].fillna(0).values
y = (merged["disparity_score"] > merged["disparity_score"].median()).astype(int).values
 
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
 
X_train, X_test, y_train, y_test = train_test_split(X_scaled, y, test_size=0.2, random_state=42)
 
model = LogisticRegression(random_state=42)
model.fit(X_train, y_train)
 
merged["predicted_high"] = model.predict(X_scaled)
acc = model.score(X_test, y_test)
print(f"Accuracy: {acc:.1%}")
```

    Accuracy: 98.1%


# Analysis Rationale

To identify inequity in FEMA assistance, I computed three gap metrics per zip code: an approval gap, award gap, and damage gap for homeowners vs. renters. Respectively, this was how much more often owners got approved than renters, how much larger owner awards were, and the difference in damage per registration. I used logistic regression to classify each zip code as high or low disparity, using the median disparity scores as a threshold. Features were standardized before fitting the model so it would fairly compare gaps measured in different units (percentages rather than dollars). The 98% accuracy tells us that these three gaps are highly predictive of overall disparity.

# Visualization


```python
#scatter of owner vs renter approval rate, colored by model prediction

#points below the dashed line are zip codes where renters get approved less than owners
fig, ax = plt.subplots(figsize=(7, 6))

colors = merged["predicted_high"].map({1: "#C0392B", 0: "#2980B9"})
ax.scatter(
    merged["owner_approval_rate"],
    merged["renter_approval_rate"],
    c=colors, s=55, alpha=0.75, edgecolors="white", linewidths=0.4
)

#above dashed parity line are where renters are approved more, below is owners approved more
lim = max(merged["owner_approval_rate"].max(), merged["renter_approval_rate"].max()) + 0.05
ax.plot([0, lim], [0, lim], color="#aaaaaa", linestyle="--", linewidth=1)
ax.text(lim - 0.02, lim + 0.005, "parity", fontsize=8, color="#aaaaaa", ha="right")

ax.set_xlabel("Owner approval rate", fontsize=11)
ax.set_ylabel("Renter approval rate", fontsize=11)
ax.set_title(f"FEMA assistance: predicted disparity by ZIP code  (accuracy {acc:.0%})", fontsize=11, fontweight="bold")
ax.xaxis.set_major_formatter(plt.FuncFormatter(lambda v, _: f"{v:.0%}"))
ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda v, _: f"{v:.0%}"))

ax.legend(handles=[
    mpatches.Patch(color="#C0392B", label="High disparity (predicted)"),
    mpatches.Patch(color="#2980B9", label="Low disparity (predicted)"),
], fontsize=9, framealpha=0.5)

sns.despine()
plt.tight_layout()
plt.savefig("output/fema_equity_figure.png", dpi=300, bbox_inches="tight")
plt.show()
```


    
![png](Pipeline_files/Pipeline_14_0.png)
    


# Visualization Rationale

The scatter plot puts owner approval rate on the x axis and renter approval on the y axis. The dashed line shows where both groups would be approved equally. Points below that line are zip codes where renters get approved less than owners. This shows red zip codes as high disparity and blue as low disparity, based on a combination of approval gap, award gap, and damage gap. The model achieved 98% accuracy on the test set, meaning these three features almost perfectly separate the two groups. The clustering of red points in the upper right highlights the zip codes where the inequity is most extreme.

In practice, a tool like this could help FEMA or researchers quickly flag zip codes that need extra support. You could run a similar model after a disaster and learn areas where renters are being underserved. This could lead to outreach to renters who were denied and a review of processes that may be creating barriers. With enough development, this tool could help predict which areas are likely to show disparity before assistance goes out as a more proactive resouce allocation approach.
