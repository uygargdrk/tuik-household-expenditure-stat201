"""
TUİK Household Consumption Expenditure - STAT201 (GitHub cleaned version)

Usage:
  python src/analysis.py --data_path data/stat.xlsx --save_plots images

Notes:
- The raw dataset is not included in the repo. Download it from TUİK and put it under /data.
"""
from __future__ import annotations

import argparse
from pathlib import Path
import re

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats


COLUMNS = [
    "Survey_year",
    "Number_of_household",
    "Total",
    "Food_and_non-alcoholic_beverages",
    "Alcoholic_beverages_tobacco",
    "Clothing_and_footwear",
    "Housing_and_rent",
    "Furniture_household_appliances",
    "Health",
    "Transportation",
    "Communication",
    "Entertainment_and_culture",
    "Educational_services",
    "Restaurants_and_hotels",
    "Various_goods_and_services",
    "Insurance_and_financial_services",
    "Personal_care_miscellaneous_goods",
]


def parse_year_to_numeric(series: pd.Series) -> pd.Series:
    """Extract a 4-digit year from messy strings, return numeric year."""
    s = series.astype(str)
    year = s.str.extract(r"(\d{4})", expand=False)
    return pd.to_numeric(year, errors="coerce")


def coerce_numeric_percent(series: pd.Series) -> pd.Series:
    """Convert strings with comma decimal separators to float."""
    s = series.astype(str).str.replace(",", ".", regex=False)
    return pd.to_numeric(s, errors="coerce")


def calculate_confidence_interval(x: pd.Series, confidence_level: float = 0.95):
    """t-based CI for mean."""
    cleaned = x.dropna()
    n = len(cleaned)
    if n < 2:
        return None
    mean = cleaned.mean()
    std = cleaned.std(ddof=1)
    se = std / np.sqrt(n)
    lo, hi = stats.t.interval(confidence_level, n - 1, loc=mean, scale=se)
    return lo, hi


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--data_path", type=str, default="data/stat.xlsx", help="Path to Excel file")
    parser.add_argument("--save_plots", type=str, default="", help="Directory to save plots (optional)")
    args = parser.parse_args()

    data_path = Path(args.data_path)
    if not data_path.exists():
        raise FileNotFoundError(
            f"Data file not found: {data_path}. Put the downloaded TUİK Excel file under /data."
        )

    # Read: the TUİK table usually has header/footer rows -> skiprows/skipfooter may be needed.
    # Adjust these if TUİK updates the layout.
    df = pd.read_excel(data_path, skiprows=3, skipfooter=8)
    df.columns = COLUMNS[: len(df.columns)]  # be robust if TUİK changes columns slightly

    # Year
    df["Survey_year_numeric"] = parse_year_to_numeric(df["Survey_year"])

    # Numeric conversions (all expenditure columns except year & household count)
    expenditure_cols = [c for c in df.columns if c not in ("Survey_year", "Survey_year_numeric", "Number_of_household")]
    for col in expenditure_cols:
        df[col] = coerce_numeric_percent(df[col])

    print("\n=== Head ===")
    print(df.head())

    print("\n=== Describe ===")
    print(df[expenditure_cols].describe())

    # Confidence intervals
    print("\n=== 95% CI for mean (t-interval) ===")
    for col in expenditure_cols:
        ci = calculate_confidence_interval(df[col])
        if ci is None:
            print(f"{col}: insufficient data")
        else:
            print(f"{col}: ({ci[0]:.2f}, {ci[1]:.2f})")

    # Plots
    out_dir = Path(args.save_plots) if args.save_plots else None
    if out_dir:
        out_dir.mkdir(parents=True, exist_ok=True)

    for col in expenditure_cols:
        # Histogram
        plt.figure(figsize=(10, 6))
        sns.histplot(df[col].dropna(), kde=True)
        plt.title(f"Distribution of {col} (%)")
        plt.xlabel("Expenditure share (%)")
        plt.ylabel("Frequency")
        if out_dir:
            plt.savefig(out_dir / f"hist_{col}.png", dpi=200, bbox_inches="tight")
        else:
            plt.show()
        plt.close()

        # Boxplot
        plt.figure(figsize=(8, 6))
        sns.boxplot(y=df[col].dropna())
        plt.title(f"Box Plot of {col} (%)")
        plt.ylabel("Expenditure share (%)")
        if out_dir:
            plt.savefig(out_dir / f"box_{col}.png", dpi=200, bbox_inches="tight")
        else:
            plt.show()
        plt.close()

        # Trend over time (if year exists)
        if df["Survey_year_numeric"].notna().any():
            plt.figure(figsize=(10, 6))
            sns.scatterplot(x=df["Survey_year_numeric"], y=df[col])
            plt.title(f"{col} (%) over time")
            plt.xlabel("Survey year")
            plt.ylabel("Expenditure share (%)")
            plt.grid(True)
            if out_dir:
                plt.savefig(out_dir / f"trend_{col}.png", dpi=200, bbox_inches="tight")
            else:
                plt.show()
            plt.close()

    # Simple probability-style question example
    if "Housing_and_rent" in df.columns:
        valid = df["Housing_and_rent"].dropna()
        if len(valid) > 0:
            p = (valid > 26).mean()
            print(f"\nP(Housing_and_rent > 26%) ≈ {p:.2f}")

    print("\nDone.")


if __name__ == "__main__":
    main()
