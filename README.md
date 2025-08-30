
# Recency, Frequency and Monetary (RFM) Analysis in Python

Many businesses stumble by treating all customers the same, leaning on generic marketing tactics and uniform messaging, yet struggle with flat revenue growth. The reality is, customers differ greatly in their value and behavior. RFM Analysis offers a clear, data-driven approach to identify your most valuable customers and decode their purchasing patterns.
In this project, I worked on a practical, step-by-step implementation of RFM Analysis using Python, equipping business to design targeted retention strategies and supercharge revenue growth.



<img width="1812" height="888" alt="Screenshot 2025-08-09 084115" src="https://github.com/user-attachments/assets/63c04d4d-63e8-49fc-9f50-92842a52b971" />


## Dataset Description
                      Online Retail Dataset, Transactions of a UK-based non-store online retail  

 - InvoiceNo   : Unique 8-digit invoice number for each transaction. 
              Invoices starting with 'C' indicate cancellations.
 - StockCode   : Unique 8-digit code for each product.
 - Description : Name of the product.
 - Quantity    : Number of units of the product in the transaction.
 - InvoiceDate : Date and time of the transaction.
 - UnitPrice   : Price per unit (in GBP).
 - CustomerID  : Unique 9-digit customer identifier.
 - Country     : Country where the customer resides.

Notes:
- Contains transactions from 01/12/2022 to 09/12/2024 for a UK-based online retailer.
- The company sells unique, all-occasion gifts.
- Many customers are wholesalers.


 ## What is RFM Analysis?

RFM stands for Recency, Frequency, and Monetary. It’s a marketing analysis tool used to rank quantitatively and segment customers based on their purchasing habits. Here’s how it helps:
   
 - Recency (R): How recently did a customer make a purchase? The more recent their purchase, the more likely they are to respond to promotions.
 - Frequency (F): How often do they buy? Customers who buy more frequently are generally more engaged and loyal.
 - Monetary (M): How much do they spend? Customers who spend more money are often your most profitable.

By combining these three metrics, you can group customers into different segments and develop targeted strategies for each group.


## Import the necessary libraries

```
import pandas as pd
from datetime import datetime as dt, timedelta
import plotly.express as px
import plotly.graph_objects as go
import plotly.colors

```

## Load and inspect the dataset

```
data = pd.read_csv(r"C:\Users\phabr\Downloads\retail_csv.csv")

print(" Dataset Loaded")
print(data.head())       # Display first 5 rows
print(data.tail())       # Display last 5 rows
print(data.info())       # Data types + missing values
print(data.describe())   # Numerical statistics

```

## Data Cleaning and Preparation

```
# Remove rows with missing CustomerID
data.dropna(subset=['CustomerID'], inplace=True)

# Convert InvoiceDate to datetime
data['InvoiceDate'] = pd.to_datetime(data['InvoiceDate'])

# Remove duplicate InvoiceNo entries (if any)
data.drop_duplicates(subset=['InvoiceNo'], inplace=True)

# Calculate total amount per transaction
data['TotalAmount'] = data['Quantity'] * data['UnitPrice']

print("\n After cleaning:")
print(data.info())

# Reference Date for Recency Calculation
# If data were live, reference date = today; here, we take the day after last transaction
reference_date = data['InvoiceDate'].max() + timedelta(days=1)

```

## Create RFM Table

Calculate RFM Values.
To perform the analysis, we calculate Recency, Frequency, and Monetary values for each customer.  

 - Recency: Set a snapshot date (latest date in the dataset plus one day). Subtract the customer’s most recent purchase date to get the days since their last purchase.  
 - Frequency: Count the total number of orders per customer.  
 - Monetary: Sum the total transaction amounts for each customer.


```
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

```
## Define R, F, M scoring functions

Now, we’ll transform the raw RFM values into scores (1-4) to create customer segments. The scoring is based on quartiles:

  - Recency: A lower Recency value (meaning a more recent purchase) gets a higher score. So, customers in the first quartile (least recent) get a score of 1, while those in the last quartile (most recent) get a score of 4.
  - Frequency and Monetary: The logic is reversed. A higher Frequency or Monetary value gets a higher score. Customers in the first quartile get a score of 1, while those in the last quartile get a score of 4.

Here’s how we can segment our customers:

```
# Calculate Quantiles

quantiles = rfm.quantile(q=[0.25, 0.50, 0.75])

def RScore(x, p, d):
    # Assign R, F, or M score based on quantiles
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

```

## Assign segment labels (High, Mid, Low value customers)

```
def assign_segment(score):
    #Label customers based on RFM score
    if score < 5:
        return 'Low Value'
    elif score < 9:
        return 'Mid Value'
    else:
        return 'High Value'

rfm['RFM_Segment_Labels'] = rfm['RFM_Score'].apply(assign_segment)

print("\n RFM Segment Labels Assigned")
print(rfm.head())

```
# Count customers in each segment and Visualization - Bar Chart

```
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

```
<img width="1888" height="904" alt="Screenshot 2025-08-09 084143" src="https://github.com/user-attachments/assets/8d9d9973-5992-4729-b60d-5d16813e373f" />


Based on the provided customer segmentation analysis, the customers have been categorized into three segments:

 - High-Value Customers (1692, 33.84 % of the total)

These customers are the backbone of revenue, likely contributing the majority of profits (per the Pareto principle, ~20% of customers often drive ~80% of revenue).
They are loyal, engaged, and likely to respond well to upselling, cross-selling, or premium offerings (e.g., subscriptions, exclusive products).
Retention is critical: Losing these customers would significantly impact revenue.

Example of Impact: If a high-value customer spends $1,000 monthly and stops purchasing, the business could lose $12,000 annually per customer.

 - Medium-Value Customers (2611, 52.22 % of the total)

These customers are valuable but not as critical as high-value customers. They have potential to become high-value with the right engagement.
Their slightly lower recency suggests they may need nudging to purchase more regularly.
They contribute significantly to revenue but are more price-sensitive or less loyal than high-value customers.

Example Impact: A medium-value customer spending $300 every two months could be nudged to spend $400 monthly with targeted promotions, 
increasing annual revenue per customer from $1,800 to $4,800.

 - Low-Value Customers (697, 14.38 % of the total)
 
These customers contribute the least to revenue and may include one-time or sporadic buyers.
They are at high risk of churn and may not be worth heavy investment unless they show potential to move to a higher-value segment.
Acquiring new low-value customers is often more expensive than retaining or upgrading existing ones, so focus on cost-effective engagement.

Example Impact: A low-value customer spending $50 once a year may cost more to re-engage (e.g., $10 in marketing) than their contribution, so campaigns should be low-cost and scalable.

  
# Assign descriptive customer segments
```
rfm['RFM_Customer_Segment'] = ''  # Initialize

rfm.loc[rfm['RFM_Score'] >= 9, 'RFM_Customer_Segment'] = 'VIP/Loyal'
rfm.loc[(rfm['RFM_Score'] >= 6) & (rfm['RFM_Score'] < 9), 'RFM_Customer_Segment'] = 'Potential Loyal'
rfm.loc[(rfm['RFM_Score'] >= 5) & (rfm['RFM_Score'] < 6), 'RFM_Customer_Segment'] = 'At Risk'
rfm.loc[(rfm['RFM_Score'] >= 4) & (rfm['RFM_Score'] < 5), 'RFM_Customer_Segment'] = "Can't Lose"
rfm.loc[(rfm['RFM_Score'] >= 3) & (rfm['RFM_Score'] < 4), 'RFM_Customer_Segment'] = 'Lost'

print("\n Descriptive Segments Assigned:")
print(rfm[['RFM_Score', 'RFM_Customer_Segment']].head(10))

```
Now, we assign each customer to a segment based on their combined score using a defined function. 
Here’s a breakdown of some key segments and their typical scores:

 - VIP/Loyal: Your best customers. They often buy and spend the most.
 - Potiential Loyal: High Frequency and monetary value, but their recency might be slightly lower than that of VIP.
 - At risk/ Can't lose: Bought recently but haven’t purchased frequently or spent much.
 - Lost Customers: Haven’t bought in a long time, don’t buy often, and don’t spend much.


## Visualization - Treemap

``` 
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

<img width="1812" height="888" alt="Screenshot 2025-08-09 084115" src="https://github.com/user-attachments/assets/9a21b6da-a8f1-4543-8f57-e69a4f9cf130" />


## Concrete Business Implications and Recommendations

 - Resource Allocation:
   
 By focusing 60–70% of marketing and retention budgets on high-value customers to maximize ROI, allocating 20–30% to medium-value customers to convert them into high-value and
using low-cost, automated strategies for low-value customers to minimize expenses.

 - Revenue Growth:

Upselling high-value customers (e.g., premium products) could increase their average spend by 10–20%. Converting 10% of medium-value customers to high-value could boost revenue significantly (e.g., $60,000 in the example above if 300 medium-value customers increase spend to $350). 


- Customer Lifetime Value (CLV):

High-value customers have the highest CLV, so retention efforts here yield the greatest long-term returns. Medium-value customers have moderate CLV but can be nurtured to increase it.

- Churn Prevention:

Monitor high-value customers for signs of reduced recency or frequency and act quickly (e.g., personalized offers). For medium-value customers, focus on improving recency through timely campaigns. For low-value customers, use churn prediction models to identify those worth saving.
















   






