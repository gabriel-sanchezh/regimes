---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.15.2
kernelspec:
  display_name: Python 3.8.5 ('base')
  language: python
  name: python3
---

# Data Preparation

+++

```stata
cd
```

+++

```python
# Import the required libraries
print('hello')
```

```{code-cell} ipython3
# Import the required libraries
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from functools import reduce
from fractions import Fraction

# Create a dictionary called 'g' that will be used to store global variables
g = globals()

# Set the plot style to 'ggplot' for consistent plot appearance
plt.style.use('ggplot')

import warnings
warnings.filterwarnings("ignore")
```

```{code-cell} ipython3
# Define lists for variables and countries
vars = ['cpi', 'm2', 'mpr', 'er', 'cons', 'ppi', 'ie']
countries = ['arg', 'bra', 'chl', 'col', 'cri', 'mex']
```

```{code-cell} ipython3
# Data Preparation - SIE Data (CPI, M2, MPR)

sie_columns = {
    '[Fecha]':'date',
    '[ARGENTINA]':'arg',
    '[BRASIL]':'bra',
    '[CHILE]':'chl',
    '[COLOMBIA]':'col',
    '[COSTA RICA]':'cri',
    '[MEXICO]':'mex'
}

sie_vars = ['cpi', 'm2', 'mpr']

# Load and preprocess SIE data for each variable
for x in sie_vars:
    g[x] = pd.read_excel(f'./data/{x}.xlsx', 
        header=1,
        usecols=sie_columns.keys()
    ).rename(columns=sie_columns
    ).set_index('date').dropna(how='all')
    g[x].index = pd.PeriodIndex(g[x].index, freq='M')
```

```{code-cell} ipython3
# Data Preparation - FRED Data (US CPI)

uscpi = pd.read_excel('./data/uscpi.xls',
        header=10
).rename(columns={'observation_date':'date', 'CPIAUCSL':'uscpi'}
).set_index('date')
uscpi.index = pd.PeriodIndex(uscpi.index, freq='M')
```

```{code-cell} ipython3
# Data Preparation - OECD Data (Exchange Rates)

# Load 'oecd_er' data
oecd_er = pd.read_csv('./data/er.csv',
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION':'country', 'TIME':'date', 'Value':'er'}
).set_index('date')

# Convert the index to a monthly PeriodIndex
oecd_er.index = pd.PeriodIndex(oecd_er.index, freq='M')

# Create an empty DataFrame 'cons'
er = pd.DataFrame()

# Extract exchange rates for specified countries
for c in countries:
    er[c] = oecd_er[oecd_er['country'] == c.upper()]['er'].dropna()
```

```{code-cell} ipython3
# Data Preparation - OECD Data (Consumption)

# Load 'oecd_cons' data
oecd_cons = pd.read_csv('./data/cons.csv',
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION':'country', 'TIME':'date', 'Value':'cons'}
).set_index('date')

# Convert the index to a monthly PeriodIndex
oecd_cons.index = pd.PeriodIndex(oecd_cons.index, freq='Q')

# Create an empty DataFrame 'cons'
cons = pd.DataFrame()

# Interpolate and populate 'cons' DataFrame for specific countries
for c in countries:
    cons[c] = oecd_cons[oecd_cons['country'] == c.upper()]['cons'].resample('M').interpolate()
```

```{code-cell} ipython3
# Data Preparation - OECD Data (Inflation Expectations)

# Load 'ie_q' data (quarterly)
ie_q = pd.read_excel('./data/ie_q.xlsx',
    header=1,
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION':'country', 'TIME':'date', 'Value':'ie'}
).set_index('date')

# Convert the index to a quarterly PeriodIndex
ie_q.index = pd.PeriodIndex(ie_q.index, freq='Q')

# Load 'ie_y' data (yearly)
ie_y = pd.read_excel('./data/ie_y.xlsx',
    header=1,
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION':'country', 'TIME':'date', 'Value':'ie'}
).set_index('date')

# Convert the index to a yearly PeriodIndex
ie_y.index = pd.PeriodIndex(ie_y.index, freq='Y')

# Create an empty DataFrame 'ie'
ie = pd.DataFrame()

# Interpolate and populate 'ie' DataFrame for specific countries
for c in ['mex', 'chl', 'col', 'cri']:
    ie[c] = ie_q[ie_q['country'] == c.upper()]['ie'].resample('M').interpolate()

for c in ['arg', 'bra']:
    ie[c] = ie_y[ie_y['country'] == c.upper()]['ie'].resample('M').interpolate()
```

```{code-cell} ipython3
# Data Preparation - Producer Prices Index (PPI)

# Load PPI data for 'arg' and 'bra'
for c in ['arg', 'bra']:
    g[f'ppi_{c}'] = pd.read_excel('./data/ppi.xlsx',
        sheet_name=c,
        usecols=['Date', 'index']
    ).rename(columns={'Date': 'date', 'index': 'ppi'}
    ).set_index('date')

    g[f'ppi_{c}'].index = pd.PeriodIndex(g[f'ppi_{c}'].index, freq='M')

# Load OECD PPI data
oecd_ppi = pd.read_csv('./data/oecd_ppi.csv',
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION': 'country', 'TIME': 'date', 'Value': 'ppi'}
).set_index('date')

oecd_ppi.index = pd.PeriodIndex(oecd_ppi.index, freq='M')

# Load OECD PPI data for 'chl'
ppi_chl = pd.read_csv('./data/oecd_ppi_chl.csv',
    usecols=['LOCATION', 'TIME', 'Value']
).rename(columns={
    'LOCATION': 'country', 'TIME': 'date', 'Value': 'ppi'}
).set_index('date')

ppi_chl.index = pd.PeriodIndex(ppi_chl.index, freq='M')

# Create an empty DataFrame 'ppi'
ppi = pd.DataFrame()

# Populate 'ppi' DataFrame for specific countries
for c in ['arg', 'bra']:
    ppi[c] = g[f'ppi_{c}']['ppi']

for c in ['col', 'cri', 'mex']:
    ppi[c] = oecd_ppi[oecd_ppi['country'] == c.upper()]['ppi']

ppi['chl'] = ppi_chl['ppi']
```

```{code-cell} ipython3
# Combine and Organize Data
data = pd.concat([cpi, m2, mpr, er, cons, ppi, ie], keys=vars)

# Create DataFrames for each country (df_)
for c in countries:
    g[f'df_{c}'] = pd.DataFrame()
    
    # Populate DataFrames with selected variables and 'uscpi'
    for v in vars:
        g[f'df_{c}'][v] = data[c][v].dropna()

    # Add 'uscpi' to the DataFrame
    g[f'df_{c}']['uscpi'] = uscpi['uscpi']
```

```{code-cell} ipython3
# Create modified DataFrames for each country (dfm_)
for c in countries:
    g[f'dfm_{c}'] = pd.DataFrame()
    
    # Calculate and populate various variables in 'dfm_'
    
    # Modification 1: 'cpi' -> Inflation shift
    g[f'dfm_{c}']['cpi'] = (
        ((g[f'df_{c}']['cpi'].dropna().pct_change(12) + 1) -
        (g[f'df_{c}']['cpi'].dropna().pct_change(36) + 1)**(Fraction(1/3))
        ) * 100
    ).dropna()
    
    # Modification 2: 'm2' -> 3-year change (%)
    g[f'dfm_{c}']['m2'] = (
        g[f'df_{c}']['m2'].dropna().pct_change(36)
    ).dropna()
    
    # Modification 3: 'mpr' -> 1-year difference
    g[f'dfm_{c}']['mpr'] = (
        g[f'df_{c}']['mpr'].dropna().diff(12)
    ).dropna()
    
    # Modification 4: 'uscpi' -> 1-year change (%)
    g[f'dfm_{c}']['uscpi'] = (
        g[f'df_{c}']['uscpi'].dropna().pct_change(12).dropna() * 100
    )
    
    # Modification 5: 'er' -> 1-year change (%)
    g[f'dfm_{c}']['er'] = (
        g[f'df_{c}']['er'].dropna().pct_change(12)
    ).dropna()
    
    # Modification 6: 'cons' -> 1-year change (%)
    g[f'dfm_{c}']['cons'] = (
        g[f'df_{c}']['cons'].dropna().pct_change(12).dropna() * 100
    )
    
    # Modification 7: 'ppi' -> 1-year change (%)
    g[f'dfm_{c}']['ppi'] = (
        g[f'df_{c}']['ppi'].dropna().pct_change(12).dropna() * 100
    )
    
    # Modification 8: 'ie' -> 1-year difference
    g[f'dfm_{c}']['ie'] = (
        g[f'df_{c}']['ie'].dropna().diff(12).dropna()
    )
    
    # Select a time range ('2020-01' to '2022-12')
    g[f'dfm_{c}'] = g[f'dfm_{c}']['2020-01':'2022-12']
```

```{code-cell} ipython3
#Final adjustments of missing values

dfm_arg['cons']['2022-08'] = dfm_arg['cons']['2022-03':'2022-07'].rolling(window=5).mean().dropna()
dfm_arg['cons']['2022-09'] = dfm_arg['cons']['2022-04':'2022-08'].rolling(window=5).mean().dropna()
dfm_arg['cons']['2022-10'] = dfm_arg['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_arg['cons']['2022-11'] = dfm_arg['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_arg['cons']['2022-12'] = dfm_arg['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()

dfm_bra['ppi'].fillna(dfm_bra['ppi'].dropna().median(), inplace=True)
dfm_bra['cons']['2022-08'] = dfm_bra['cons']['2022-03':'2022-07'].rolling(window=5).mean().dropna()
dfm_bra['cons']['2022-09'] = dfm_bra['cons']['2022-04':'2022-08'].rolling(window=5).mean().dropna()
dfm_bra['cons']['2022-10'] = dfm_bra['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_bra['cons']['2022-11'] = dfm_bra['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_bra['cons']['2022-12'] = dfm_bra['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()

dfm_col['cons']['2022-10'] = dfm_col['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_col['cons']['2022-11'] = dfm_col['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_col['cons']['2022-12'] = dfm_col['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()

dfm_cri['cons']['2022-10'] = dfm_cri['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_cri['cons']['2022-11'] = dfm_cri['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_cri['cons']['2022-12'] = dfm_cri['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()

dfm_chl['cons']['2022-10'] = dfm_chl['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_chl['cons']['2022-11'] = dfm_chl['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_chl['cons']['2022-12'] = dfm_chl['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()

dfm_mex['cons']['2022-10'] = dfm_mex['cons']['2022-05':'2022-09'].rolling(window=5).mean().dropna()
dfm_mex['cons']['2022-11'] = dfm_mex['cons']['2022-06':'2022-10'].rolling(window=5).mean().dropna()
dfm_mex['cons']['2022-12'] = dfm_mex['cons']['2022-07':'2022-11'].rolling(window=5).mean().dropna()
```

```{code-cell} ipython3
# Create an ExcelWriter object for a master dfm
with pd.ExcelWriter('./data/dfm.xlsx', engine='xlsxwriter') as writer:
    # Save each DataFrame to a separate sheet
    dfm_arg.to_excel(writer, sheet_name='dfm_arg')
    dfm_bra.to_excel(writer, sheet_name='dfm_bra')
    dfm_chl.to_excel(writer, sheet_name='dfm_chl')
    dfm_col.to_excel(writer, sheet_name='dfm_col')
    dfm_cri.to_excel(writer, sheet_name='dfm_cri')
    dfm_mex.to_excel(writer, sheet_name='dfm_mex')
```
