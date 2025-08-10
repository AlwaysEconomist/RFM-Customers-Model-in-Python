# Recency, Frequency and Monetary (RFM) Analysis in Python

<img width="1888" height="904" alt="Screenshot 2025-08-09 084143" src="https://github.com/user-attachments/assets/7f8461c3-5ca4-4032-b035-e0f871d6f149" />



```
python
"""
                     Online Retail Dataset, Transactions of a UK-based non-store online retail  

Dataset Description

InvoiceNo   : Unique 8-digit invoice number for each transaction. 
              Invoices starting with 'C' indicate cancellations.
StockCode   : Unique 8-digit code for each product.
Description : Name of the product.
Quantity    : Number of units of the product in the transaction.
InvoiceDate : Date and time of the transaction.
UnitPrice   : Price per unit (in GBP).
CustomerID  : Unique 9-digit customer identifier.
Country     : Country where the customer resides.

Notes:
- Contains transactions from 01/12/2022 to 09/12/2024 for a UK-based online retailer.
- The company sells unique, all-occasion gifts.
- Many customers are wholesalers.
"""


# ===========================================
 # RFM Customer Analysis 
 # ===========================================

# Import the necessary libraries
import pandas as pd
from datetime import datetime as dt, timedelta
import plotly.express as px
import plotly.graph_objects as go
import plotly.colors

# Load and inspect the dataset

data = pd.read_csv(r"C:\Users\phabr\Downloads\retail_csv.csv")

print("🔹 Dataset Loaded")
print(data.head())       # Display first 5 rows
print(data.tail())       # Display last 5 rows
print(data.info())       # Data types + missing values
print(data.describe())   # Numerical statistics


# Data Cleaning

# Remove rows with missing CustomerID
data.dropna(subset=['CustomerID'], inplace=True)

# Convert InvoiceDate to datetime
data['InvoiceDate'] = pd.to_datetime(data['InvoiceDate'])

# Remove duplicate InvoiceNo entries (if any)
data.drop_duplicates(subset=['InvoiceNo'], inplace=True)

# Calculate total amount per transaction
data['TotalAmount'] = data['Quantity'] * data['UnitPrice']

print("\n✅ After cleaning:")
print(data.info())


# Reference Date for Recency Calculation
# If data were live, reference date = today; here, we take the day after last transaction
reference_date = data['InvoiceDate'].max() + timedelta(days=1)


# Create RFM Table
rfm = data.groupby('CustomerID').agg({
    'InvoiceDate': lambda x: (reference_date - x.max()).days,  # Recency: days since last purchase
    'InvoiceNo': 'count',                                      # Frequency: number of purchases
    'TotalAmount': 'sum'                                       # Monetary: total spend
})

# Rename columns for clarity
rfm.rename(
    columns={
        'InvoiceDate': 'Recency',
        'InvoiceNo': 'Frequency',
        'TotalAmount': 'Monetary'
    },
    inplace=True
)

# Calculate Quantiles

quantiles = rfm.quantile(q=[0.25, 0.50, 0.75])


# Define R, F, M scoring functions

def RScore(x, p, d):
    """Assign R, F, or M score based on quantiles"""
    if p == 'Recency':  # Lower recency = better score
        if x <= d[p][0.25]:
            return 4
        elif x <= d[p][0.50]:
            return 3
        elif x <= d[p][0.75]:
            return 2
        else:
            return 1
    else:  # Higher F and M = better score
        if x <= d[p][0.25]:
            return 1
        elif x <= d[p][0.50]:
            return 2
        elif x <= d[p][0.75]:
            return 3
        else:
            return 4


# Apply R, F, M scoring

rfm['R'] = rfm['Recency'].apply(RScore, args=('Recency', quantiles,))
rfm['F'] = rfm['Frequency'].apply(RScore, args=('Frequency', quantiles,))
rfm['M'] = rfm['Monetary'].apply(RScore, args=('Monetary', quantiles,))

# Create combined segment & score
rfm['RFM_Segment'] = rfm['R'].astype(str) + rfm['F'].astype(str) + rfm['M'].astype(str)
rfm['RFM_Score'] = rfm[['R', 'F', 'M']].sum(axis=1)


# Assign segment labels (High, Mid, Low value customers)

def assign_segment(score):
    #Label customers based on RFM score
    if score < 5:
        return 'Low Value'
    elif score < 9:
        return 'Mid Value'
    else:
        return 'High Value'

rfm['RFM_Segment_Labels'] = rfm['RFM_Score'].apply(assign_segment)

print("\n🏷️ RFM Segment Labels Assigned")
print(rfm.head())

# Assign descriptive customer segments

rfm['RFM_Customer_Segment'] = ''  # Initialize

rfm.loc[rfm['RFM_Score'] >= 9, 'RFM_Customer_Segment'] = 'VIP/Loyal'
rfm.loc[(rfm['RFM_Score'] >= 6) & (rfm['RFM_Score'] < 9), 'RFM_Customer_Segment'] = 'Potential Loyal'
rfm.loc[(rfm['RFM_Score'] >= 5) & (rfm['RFM_Score'] < 6), 'RFM_Customer_Segment'] = 'At Risk'
rfm.loc[(rfm['RFM_Score'] >= 4) & (rfm['RFM_Score'] < 5), 'RFM_Customer_Segment'] = "Can't Lose"
rfm.loc[(rfm['RFM_Score'] >= 3) & (rfm['RFM_Score'] < 4), 'RFM_Customer_Segment'] = 'Lost'

print("\n📌 Descriptive Segments Assigned:")
print(rfm[['RFM_Score', 'RFM_Customer_Segment']].head(10))


# Count customers in each segment and Visualization - Bar Chart

segment_counts = rfm['RFM_Segment_Labels'].value_counts().reset_index()
segment_counts.columns = ['RFM_Segment', 'Count']
segment_counts = segment_counts.sort_values('RFM_Segment')

fig_bar = px.bar(
    segment_counts,
    x='RFM_Segment',
    y='Count',
    title='Customer Distribution by RFM Segment',
    labels={'RFM_Segment': 'RFM Segment', 'Count': 'Number of Customers'},
    color='RFM_Segment',
    color_discrete_sequence=px.colors.qualitative.Pastel
)
fig_bar.show()


# Visualization - Treemap
 
segment_product_counts = (
    rfm.groupby(['RFM_Segment_Labels', 'RFM_Customer_Segment'])
       .size()
       .reset_index(name='Count')
       .sort_values('Count', ascending=False)
)

fig_treemap = px.treemap(
    segment_product_counts,
    path=['RFM_Segment_Labels', 'RFM_Customer_Segment'],
    values='Count',
    color='RFM_Segment_Labels',
    color_discrete_sequence=px.colors.qualitative.Pastel,
    title='RFM Customer Segments by Value'
)
fig_treemap.show()





```
<img width="1812" height="888" alt="Screenshot 2025-08-09 084115" src="https://github.com/user-attachments/assets/8272931a-62aa-47f5-8049-2ffddb6ee228" />

