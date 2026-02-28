{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "e0449140",
   "metadata": {},
   "source": [
    "# TUİK Household Consumption Expenditure (STAT201)\n",
    "\n",
    "This notebook runs the same analysis as `src/analysis.py` but interactively."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "99f6a09f",
   "metadata": {},
   "outputs": [],
   "source": [
    "import pandas as pd\n",
    "import numpy as np\n",
    "import matplotlib.pyplot as plt\n",
    "import seaborn as sns\n",
    "from scipy import stats\n",
    "from pathlib import Path\n"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "9ce1f0a6",
   "metadata": {},
   "source": [
    "## 1) Load data\n",
    "\n",
    "Put your Excel file under `data/stat.xlsx`."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "92be07fb",
   "metadata": {},
   "outputs": [],
   "source": [
    "data_path = Path('data/stat.xlsx')\n",
    "assert data_path.exists(), 'Missing data/stat.xlsx (download it from TUİK first)'\n",
    "df = pd.read_excel(data_path, skiprows=3, skipfooter=8)\n",
    "df.head()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "9d8206a7",
   "metadata": {},
   "source": [
    "## 2) Basic cleaning"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "9055b2f3",
   "metadata": {},
   "outputs": [],
   "source": [
    "df.columns = [\n",
    "    'Survey_year','Number_of_household','Total','Food_and_non-alcoholic_beverages',\n",
    "    'Alcoholic_beverages_tobacco','Clothing_and_footwear','Housing_and_rent',\n",
    "    'Furniture_household_appliances','Health','Transportation','Communication',\n",
    "    'Entertainment_and_culture','Educational_services','Restaurants_and_hotels',\n",
    "    'Various_goods_and_services','Insurance_and_financial_services','Personal_care_miscellaneous_goods'\n",
    "][:len(df.columns)]\n",
    "\n",
    "df['Survey_year_numeric'] = pd.to_numeric(df['Survey_year'].astype(str).str.extract(r'(\\d{4})')[0], errors='coerce')\n",
    "expenditure_cols = [c for c in df.columns if c not in ('Survey_year','Survey_year_numeric','Number_of_household')]\n",
    "for c in expenditure_cols:\n",
    "    df[c] = pd.to_numeric(df[c].astype(str).str.replace(',', '.', regex=False), errors='coerce')\n",
    "\n",
    "df[expenditure_cols].describe()"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "d1745b61",
   "metadata": {},
   "source": [
    "## 3) Confidence intervals"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2ff1d258",
   "metadata": {},
   "outputs": [],
   "source": [
    "def ci_mean(x, level=0.95):\n",
    "    x = x.dropna()\n",
    "    n = len(x)\n",
    "    if n < 2:\n",
    "        return None\n",
    "    mean = x.mean()\n",
    "    se = x.std(ddof=1) / np.sqrt(n)\n",
    "    return stats.t.interval(level, n-1, loc=mean, scale=se)\n",
    "\n",
    "for c in expenditure_cols:\n",
    "    ci = ci_mean(df[c])\n",
    "    if ci:\n",
    "        print(f'{c}: ({ci[0]:.2f}, {ci[1]:.2f})')"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "037a514e",
   "metadata": {},
   "source": [
    "## 4) Example plots"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "cd8b4b70",
   "metadata": {},
   "outputs": [],
   "source": [
    "col = 'Housing_and_rent'\n",
    "sns.scatterplot(x=df['Survey_year_numeric'], y=df[col])\n",
    "plt.title(f'{col} over time')\n",
    "plt.grid(True)\n",
    "plt.show()"
   ]
  }
 ],
 "metadata": {},
 "nbformat": 4,
 "nbformat_minor": 5
}
