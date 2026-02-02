# PSID_2026
Analysis of parental homeownership effects on child educational attainment using PSID data

# Parental Homeownership and Child Educational Attainment

Analysis using Panel Study of Income Dynamics (PSID) data examining whether parental homeownership in 1968 is associated with children's educational attainment.

## Research Question
Do children whose parents owned their home in 1968 achieve higher educational attainment compared to children whose parents rented?

## Key Finding
Children of homeowners completed approximately 0.9 more years of education (p < 0.001), controlling for race and sex.

## Repository Structure
- `data_preparation/`: Notebooks for linking generations and preparing variables
- `exploration/`: Exploratory data analysis
- `analysis/`: Main regression analyses
- `outputs/`: Generated figures and tables

## About This Book
This code accompanies my book [insert GitBook link], which documents the journey from curiosity to analysis.

## Data Source
Panel Study of Income Dynamics (PSID): https://psidonline.isr.umich.edu/

## Requirements
- Python 3.x
- pandas
- statsmodels
- matplotlib
- Google Colab (for easy reproduction)

## Citation
[Your name]. (2026). *Analysis of Parental Homeownership Effects on Child Education*.

## Contact
[JFoley648@gmail.com]
```

### 5. Link from Your GitBook

In your GitBook, add something like:

> "All code for this analysis is available on GitHub at [github.com/yourusername/psid-homeownership-education](link). Feel free to explore, reproduce, or extend the analysis."

## Important Considerations

**Data Privacy:**
- Don't upload the actual PSID data files (they're restricted)
- Only upload your code/notebooks
- Add a note in README about how others can access PSID data

**Add a .gitignore file:**
```
# Data files (don't upload restricted PSID data)
*.csv
*.dta
*.sav
data/

# Google Drive artifacts
.google/
