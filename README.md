# RFM Segmentation (Python & Pandas)

## Purpose
Turning clean transactions into one scored row per customer, assigns a segment and a churn (lost Customer) flag, then merges those fields back onto the transaction table for Power BI.

Environment: Jupyter notebook (Python 3, pandas).

Load the library and data
python
import pandas as pd

cols = ['Invoice', 'StockCode', 'Description', 'Quantity', 'InvoiceDate',
        'Price', 'Customer_ID', 'Country', 'SourceSheet']

df = pd.read_csv('Main_Data.csv', header=None, names=cols,
                 dtype={'Invoice': str, 'StockCode': str}, low_memory=False)

df['InvoiceDate'] = pd.to_datetime(df['InvoiceDate'])
print(df.shape)     # (1033036, 9)
df.info()

<img width="1348" height="684" alt="image" src="https://github.com/user-attachments/assets/c98dbb44-9080-47c0-9807-7a4d949eb516" />

## Filter and calculate Recency, Frequency, Monetary
python
rfm_data = df[(df['Customer_ID'].notna()) & (df['Quantity'] > 0)].copy()
snapshot_date = pd.to_datetime('2011-12-10')

rfm = rfm_data.groupby('Customer_ID').agg(
    Recency=('InvoiceDate', lambda x: (snapshot_date - x.max()).days),
    Frequency=('Invoice', 'nunique'),
    Monetary=('Quantity', lambda x: (x * rfm_data.loc[x.index, 'Price']).sum())
).reset_index()
Rule	Why
Customer_ID.notna()	A score needs a customer
Quantity > 0	Removes returns and adjustments
snapshot_date = 2011-12-10	One day after the last transaction in the data, so Recency is measured "as of today"
Recency	Days since the customer's most recent invoice (lower is better)
Frequency	Number of unique invoices (not rows, so a 10-line basket counts once)
Monetary	Sum of Quantity × Price

## Score customers (1-5) and assign segments
python
rfm['R_Score'] = pd.qcut(rfm['Recency'], 5, labels=[5, 4, 3, 2, 1]).astype(int)
rfm['F_Score'] = pd.qcut(rfm['Frequency'].rank(method='first'), 5,
                         labels=[1, 2, 3, 4, 5]).astype(int)
rfm['M_Score'] = pd.qcut(rfm['Monetary'], 5, labels=[1, 2, 3, 4, 5]).astype(int)
Recency labels are reversed (5 → 1) because a smaller number of days is better.
Frequency uses rank(method='first') because many customers share the same low frequency (1 or 2 orders). Without the rank step, qcut fails with duplicate bin edges.
python
def segment_customer(row):
    r, f, m = row['R_Score'], row['F_Score'], row['M_Score']
    if r >= 4 and f >= 4 and m >= 4:
        return 'Champions'
    elif r >= 3 and f >= 3:
        return 'Loyal'
    elif r <= 2 and f >= 3:
        return 'At Risk'
    elif r >= 4 and f <= 2:
        return 'New'
    elif r == 3 and f <= 2:
        return 'Needs Attention'
    else:
        return 'Lost'

rfm['Segment'] = rfm.apply(segment_customer, axis=1)
rfm['IsChurned'] = rfm['Recency'] > 90
Segment	Rule (R, F, M scores)	Meaning
Champions	R ≥ 4, F ≥ 4, M ≥ 4	Recent, frequent, high spenders
Loyal	R ≥ 3, F ≥ 3	Buy regularly
At Risk	R ≤ 2, F ≥ 3	Used to buy often but have gone quiet
New	R ≥ 4, F ≤ 2	Recent, few orders
Needs Attention	R = 3, F ≤ 2	Middling recency, low frequency
Lost	Remaining (low R, low F)	Inactive and low engagement

Churn definition: IsChurned = Recency > 90 days.

<img width="1365" height="691" alt="image" src="https://github.com/user-attachments/assets/1e0bae21-53fd-48e7-ad3c-d3af52895609" />

## Merge RFM results back onto transactions and export
python
df = df[(df['Customer_ID'].notna()) & (df['Quantity'] > 0)].copy()
df['InvoiceMonth'] = df['InvoiceDate'].dt.to_period('M').dt.to_timestamp()
df['Revenue'] = df['Quantity'] * df['Price']

df_final = df.merge(
    rfm[['Customer_ID', 'Recency', 'Frequency', 'Monetary',
         'R_Score', 'F_Score', 'M_Score', 'Segment', 'IsChurned']],
    on='Customer_ID', how='left', validate='many_to_one'
)

df_final.to_excel('EcommerceData.xlsx', index=False)

<img width="1365" height="689" alt="image" src="https://github.com/user-attachments/assets/9f3522bb-facb-4c2f-904d-1c18250ddd69" />

## Output schema (19 columns)
1	Invoice	Transaction	11	Revenue	Transaction
2	StockCode	Transaction	12	Recency	Customer
3	Description	Transaction	13	Frequency	Customer
4	Quantity	Transaction	14	Monetary	Customer
5	InvoiceDate	Transaction	15	R_Score	Customer
6	Price	Transaction	16	F_Score	Customer
7	Customer_ID	Key	17	M_Score	Customer
8	Country	Customer	18	Segment	Customer
9	SourceSheet	Transaction	19	IsChurned	Customer
10	InvoiceMonth	Transaction.	
