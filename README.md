## 0. Prep S3 Directory/Bucket for Project

**Notes:** 
- Create folders in a s3 directory/bucket:
    - aggregate_data (final outputs from scripts: 03.comaprison.ipynb, 04_x.ipynb)
    - auto_detergent(01_data_no_segmentation.ipynb i.e benchmark data/categories)
    - check_point_path (stores files used during joins, not used in final outputs)
    - raw_data (data from desired categories for segmentation, written form 01.1_data_load.ipynb)
    - segmented_customers_joined (written from 03.1_comparison.ipynb)
    - segmented_customers (written from 01.3_data_segmentation)
    - shabit_segmentations_rex(manually uploaded files from REX)
    - shoprite_features(contains all member level features)
    - member_base(create in Athena to see size of member base relative to cohorts and baselines)


## 1. `01_data_no_segmentation.ipynb`

**Description:** Generates benchmark datasets used as a baseline for comparison against segmented data outputs.

**Inputs:**

| Parameter | Description |
|---|---|
| `sales_table` | sales table |
| `article_table` | article table |
| `merchandise_category_hierarchy` | Filter by merchandise category. |
| `sub_category_code` | Filter by sub-category. |
| `category_group_no` | Filter by category group. |
| `brand_id_description`  | Filter by brand description: Specify if you want the brand filtered other leave blank and all brands are selected. |
| `brand_code_operation` | Should always be 'in'. |
| `article_code_list` | List of article codes to include/exclude. Can also be empty. |
| `article_code_operation`  | `"in"` or `"not in"`, controls inclusion logic for article list |
| `end_date`  | End date for the analysis window |
| `look_back_interval`  | Number of weeks to look back from end date. |
| `N` | Number of period for split. |
| `grouping_cols` | Columns to group output by. |
| `output_location` | Path/location to write output. |

**Outputs:**
- Aggregated data at the `grouping_cols` level, filtered by the specified category, sub-category, brand, and/or article list (based on inputs).
- Printed summary views covering general data overview and key dataset characteristics.

**Notes:**
- The article list was manually constructed from a file provided by Ismail, with additions made for high-sales articles not originally included
- Can manually replace %CHECKERS% with %SHOPRITE% in data_extraction.

---

## 2. `01.1_data_load.ipynb`

**Description:** Extracts and preprocesses member-level transaction data for products. Produces a current-window summary per member alongside previous-window metrics, which serve as inputs to the downstream segmentation pipeline.

**Inputs:**

| Parameter | Description |
|---|---|
| `sales_table`  | Source sales fact table |
| `article_table`  | Source article dimension table |
| `merch_hierarchy`  | Merchandise category filter (empty = no filter) |
| `sub_category_code`  | Sub-category filter (empty = no filter) |
| `category_group_no`  | Category group filter (empty = no filter) |
| `brand_id_description`  | Brand filter pattern |
| `brand_code_operation`  | SQL operator for brand filter |
| `article_code_list` | Explicit OMO Auto Liquid article codes to include |
| `article_code_operation`  | Inclusion/exclusion logic for article list |
| `end_date`  | End date of the analysis window |
| `look_back_interval`  | Total weeks of data to extract |
| `N`  | Current window size in weeks |
| `grouping_cols`  | Aggregation granularity |
| `output_location` | S3 path for parquet outputs |

**Outputs:**

| File | Description |
|---|---|
| `df_prepped` | Member-level current-window aggregates (baskets, sales, items, weeks) |
| `product_df` | Raw extracted transaction data with date columns added |
| `prev_total_baskets` | Previous N-week basket counts per member |
| `prev_distinct_weeks` | Previous N-week active week counts per member |
| `prev_total_sales` | Previous N-week total sales per member |
| `prev_total_items` | Previous N-week total items per member |
| `prev_avg_sales` | Previous N-week avg sales per member |
| `prev_avg_items` | Previous N-week avg items per member |
| `prev_max_date` | Latest transaction date in previous window per member |

All outputs written as parquet to the S3 `output_location`.

**Prints:**
- Sample of `date_key` vs `date_key_new` conversion
- First 10 rows of `df_prepped`
- Spearman correlation matrix across current-window metrics

**Dependencies:** 

**Notes:**

- Article list was manually constructed from a file provided by Ismail and cehcked for OMO detergents, with additions made for high-sales articles not in the original list
- Scope is **Auto detergents only** - handwash detergents are excluded at the article list level
- Data is filtered to `country_code = 'ZA'` and banners containing `CHECKERS`
- Outputs from this notebook feed directly into the segmentation notebook
- Run this notebook before the segmentation notebook

## `01.3_data_segmentation.ipynb`

**Description:** Applies segmentation to member data using the preprocessed data from the extraction notebook. Classifies members into spend/frequency segments, identifies lost and new opportunities, merges all signals into a final member-level dataset, and produces summary statistics.

**Inputs:**

| Parameter | Description |
|---|---|
| `N`  | Current window size in weeks |
| `grouping_cols`  | Aggregation granularity |
| `spend_quantile` | Quantile threshold for High spend classification |
| `freq_min_weeks` | Minimum distinct weeks for High frequency classification |

All parquet inputs read from `s3://...`:

| File | Description |
|---|---|
| `df_prepped` | Member-level current-window aggregates |
| `product_df` | Raw transaction data (used for rejector/new member logic) |
| `prev_total_baskets` | Previous N-week basket counts per member |
| `prev_distinct_weeks` | Previous N-week active week counts per member |
| `prev_total_sales` | Previous N-week total sales per member |
| `prev_total_items` | Previous N-week total items per member |
| `prev_avg_sales` | Previous N-week avg sales per member |
| `prev_avg_items` | Previous N-week avg items per member |
| `prev_max_date` | Latest transaction date in previous window per member |

**Segmentation Logic:**

Customer segments are assigned based on two dimensions:

| Spend | Frequency | Segment |
|---|---|---|
| High (≥ p75) | High (≥ 2 weeks) | `brand_advocates` |
| Low | High | `potential_advocates` |
| High | Low | `potential_advocates` |
| Low | Low | `casual_observers` |

Additionally:
- **`lost_opportunities`** - members active in the previous window but absent in the current window
- **`lost_potential`** - lost opportunity members whose previous `total_items` exceeded the `brand_advocates` average (i.e. high-value lapsed members)
- **`new_opportunities`** - members active in the current window but absent in the previous window

**Outputs:**

| Column | Description |
|---|---|
| `brand_segment` | Final customer segment per member |
| `new_old` | Whether member is new, both periods, lost opportunity, or lost potential |
| `spend_level` / `freq_level` | Intermediate spend and frequency classifications |
| `prev_*` columns | Previous window metrics joined per member |
| `lost_opportunity_expanded` | Refined lost opportunity classification |

Final output written as a single CSV to `s3://...`.

**Prints/Side Effects:**
- Segment count breakdown with % within segment and % across all members
- Segment × new/old breakdown with within-group and across-all percentages
- Segment summary table (customer count, total sales, avg active weeks, avg items, % customers, % sales)
- Duplicate member ID check (should be empty)
- First 5 rows of `df_final`

**Notes:**
- `lost_potential` is defined relative to the mean `total_items` of `brand_advocates` - this threshold shifts depending on the data, so results are not static across runs
- Output is coalesced to a single CSV partition - not suitable for very large member populations

-- 

## `02_shoprite_features.ipynb`

**Description:** Builds a enriched member-level feature table for all Checkers members by joining demographic, behavioural and propensity data from multiple source tables. Also appends an existing Checkers SHABIT segmentation from pre-computed CSVs. The output serves as a contextual enrichment layer to profile OMO segments against the broader Checkers membership base.

**Inputs:**

| Parameter | Table/Path | Description |
|---|---|---|
| `sales_table` | `prod_de_vault.fact_customer_ticket` | Used to derive the Checkers member universe |
| `parent_table` | `dev_srx_analytics...new_parent_model` | Parental status and child age estimates |
| `age_gender_table` | `prod_de_vault.dim_sap_commerce_customerprofile` | DOB and ID prefix for age/gender derivation |
| `pet_table` | `dev_srx_analytics...pet_ownership` | Pet ownership records |
| `spend_propensity_table` | `prod_data_ecosystems.affluence_segmentation_groups` | Affluence/spend propensity segments |
| `cshi_table` | `dev_srx_analytics...cshi_member_results_enriched` | Shopping Health Index scores |
| `lapsing` | `s3://...shabit_segmentations_rex/.../` | Pre-computed lapsing segment  |
| `opportunity` | `s3://...shabit_segmentations_rex/.../` | Pre-computed opportunity segment  |
| `premium` | `s3://...shabit_segmentations_rex/.../` | Pre-computed premium segment  |
| `valuable` | `s3://...shabit_segmentations_rex/.../` | Pre-computed valuable segment  |

**Feature Construction:**

| Feature Group | Columns |
|---|---|
| Parental status | `is_parent`, `child_age_band`, `total_children`, `total_children_count` | 
| Gender | `gender`, `is_male`, `is_female` | 
| Generation | `generation_classification`, `is_gen_z`, `is_millenial`, `is_generation_x`, `is_baby_boomers`, `is_silent_generation` |
| Pet ownership | `is_pet`, `is_dog`, `is_cat`, `total_pets`, `total_dogs`, `total_cats`, `*_group` buckets | 
| Spend propensity | `affluence_cat`, `affluence_cat_rank`, `is_spend_prop_*` (6 bands) | 
| CSHI | `cshi`, `cshi_decile_band`, `real_spend`, `segmentation`, `diversity`, `discretionary_spend`, `is_cshi_*` | 
| SHABIT | `shabit_segmentation` (lapsing / opportunity / premium / valuable) |

**Outputs:**
- `shoprite_features` written to `s3://...shoprite_features`
- `df_final` — `shoprite_features` left-joined with `checkers_shabit_segmentation` and `shoprite_shabit_segmentation`

**Prints/Side Effects:**
- First 5 rows of `shoprite_features`
- First row of each SHABIT CSV on load
- Duplicate `member_id` check with total rows, distinct members, and duplicate count

**Notes:**
- Members with `member_id` starting with `+27` are filtered out from `shoprite_features` before output
- All source tables are filtered to their latest `process_date` / `date_key` to ensure point-in-time consistency
- The SHABIT CSVs use `Household_Identifier` which is renamed to `member_id` on load
- All demographic joins are `left` - missing data results in `0` flags rather than dropped members
- Generation classification is derived from year of birth; members outside defined ranges will have `null` for `generation_classification`


## 02_1_member_base.ipynb

Builds the Checkers member base by joining transactional member data with Shoprite feature data, computes demographic and behavioural member counts across a set of features, and writes the aggregated output to S3.

---

#### Input Datasets

Both inputs are read from S3 as CSV files with inferred schema.

| Dataset | S3 Path | Description |
|---|---|---|
| `shoprite_features` | `s3://.../omo_tests/shoprite_features/` | Member-level demographic and behavioural feature enrichment |
| `member_base` | `s3://.../omo_tests/member_base/` | Checkers transactional member base (derived from Athena - see below) |

> **Member base Athena query** - filters to active Checkers members over the 27-week period ending 2026-03-31:
> ```sql
> SELECT
>     b.member_id,
>     COUNT(DISTINCT b.unique_ticket_id) AS total_shops
> FROM prod_de_vault.fact_customer_ticket b
> INNER JOIN prod_de_vault.dim_sap_article c
>     ON b.article_code = c.article_code
>     AND b.alternate_uom = c.alternate_uom
>     AND c.key_effective_to IS NULL
> WHERE b.country_code = 'ZA'
>     AND b.member_id IS NOT NULL
>     AND UPPER(b.banner) LIKE '%CHECKERS%'
>     AND c.super_dept_no != '99'
>     AND item_count > 0
>     AND b.sales_amount_after_discount > 0
>     AND b.sales_amount > 0
>     AND b.ticket_end_time IS NOT NULL
>     AND date_parse(CAST(b.date_key AS VARCHAR), '%Y%m%d')
>         BETWEEN date_add('day', -27 * 7, DATE '2026-03-31') AND DATE '2026-03-31'
> GROUP BY b.member_id
> ```

---

### Steps

#### 1. Load Data
Reads `shoprite_features` and `member_base` from S3 into Spark DataFrames and registers both as temp views.

#### 2. Build Member Feature Base
Inner joins `shoprite_features` with the distinct `member_id` values from `member_base` to produce `member_feature_base_df` - the enriched Checkers member base used for all downstream aggregations.

#### 3. Define Aggregate Function
`get_feature_total` computes member counts per feature value for a given demographic dimension. For each feature it:
- Optionally deduplicates on `member_id` (controlled by `multi_row_feature` - set to `True` for features like `child_age_band` where one member can have multiple rows)
- Joins the member base against the feature reference
- Groups by feature value and counts distinct members
- Computes `pct_in_group` as each value's share of the total

Sales metric columns (`avg_sg`, `total_sales`, `total_sales_prev`, `sales_growth`) are present in the schema but set to `NULL` - this notebook sizes the member base only.

#### 4. Run Across Features
The following features are profiled across the Checkers member base (~9.7M members):

| Feature | Column | Multi-row |
|---|---|---|
| All (total base) | - | - |
| Generation | `generation_classification` | No |
| Gender | `gender` | No |
| CSHI | `cshi_decile_band` | No |
| Spend Propensity | `affluence_cat_rank` | No |
| Parents | `is_parent` | No |
| Child Age Band | `child_age_band` | Yes |
| Total Children | `total_children` | No |
| Pet | `is_pet` | No |
| Dog | `is_dog` | No |
| Cat | `is_cat` | No |
| Total Pets | `total_pets_group` | No |
| Total Dogs | `total_dogs_group` | No |
| Total Cats | `total_cats_group` | No |
| Checkers Shabit Segmentation | `checkers_shabit_segmentation` | No |

#### 5. Union & Write
All feature DataFrames are unioned into a single Spark DataFrame (`df_all_features`, 48 rows) and written to S3. The enriched member feature base is also written separately.

---

## Output

| Output | S3 Path | Description |
|---|---|---|
| `df_all_features` | `s3://.../omo_tests/aggregated_data/checkers_base/` | Aggregated member base feature counts (48 rows) |
| `member_feature_base_df` | `s3://.../omo_tests/auto_detergent/member_base/` | Enriched member base at member level |

Output schema:

| Column | Type | Description |
|---|---|---|
| `section` | string | Always `Checkers Base` |
| `segment` | string | Feature display name |
| `cohort` | string | Always `Checkers Base` |
| `feature` | string | Feature value |
| `avg_sg` | double | `NULL` - not applicable |
| `total_mems` | long | Unique member count |
| `pct_in_group` | double | Share of members within feature value |
| `total_avg_sales_vals` | double | `NULL` - not applicable |
| `total_sales` | double | `NULL` - not applicable |
| `total_sales_prev` | double | `NULL` - not applicable |
| `sales_growth` | double | `NULL` - not applicable |
---

## `03.1_comparisons.ipynb`

**Description:** Joins segmented member DataFrames across product formats (Liquid, Powder, Capsules), builds cross-format overlap flags, and profiles each segment against demographic and behavioural features from `shoprite_features`. Produces a unified reporting DataFrame used for downstream comparison and visualisation.

**Inputs:**

| Parameter | Source | Description |
|---|---|---|
| `df_omo_liquid` | Output of segmentation notebook (liquid) | Member-level SHABIT segments for OMO Auto Liquid |
| `df_omo_powder` | Output of segmentation notebook (powder) | Member-level SHABIT segments for OMO Auto Powder |
| `df_omo_capsules` | Output of segmentation notebook (capsules) | Member-level SHABIT segments for OMO Auto Capsules |
| `features_df` | `shoprite_features` | Enriched member feature table from enrichment notebook |
| `suffixes` | `['liq', 'powd', 'caps']` | Short identifiers per product format |

> Requires the segmentation notebook to have been run for all three product formats, and `02_shoprite_features.ipynb` to have been run first.

**Key Functions:**

**`join_datasets(*dataframes, suffixes)`**
- Deduplicates each input on `member_id`
- Renames metric columns with per-suffix naming (`brand_segment_liq`, `sales_value_liq`, etc.)
- Computes `sales_growth_{suffix}` per format
- Full outer joins all formats on `member_id`
- Builds cross-format membership flags per suffix:

| Flag | Meaning |
|---|---|
| `in_exclusive_{suffix}` | Member appears in this format only |
| `in_one_other_{suffix}` | Member appears in this format + exactly one other |
| `in_all_{suffix}` | Member appears in all three formats |

**`get_totals_a(df, features_df, segment_name, suffixes, ...)`**
- Per suffix, produces cohort-level summary blocks (no feature breakdown):
  - Group block - by `brand_segment` with `pct_in_group`
  - Total block - full rollup across all segments
  - Exclusive / one_other / in_all blocks - overlap cohort summaries

**`_build_group_blocks(feature_name, df, features_df, suffix, ...)`**
- Core aggregation helper used by `get_feature_data_a`
- Produces a group block (by `brand_segment` × `feature`) and a total rollup block per suffix
- Supports `multi_row_feature=True` for features like `child_age_band` where a member legitimately has multiple rows

**`get_feature_data_a(feature_name, df, features_df, segment_name, suffixes, ...)`**
- Profiles a single feature across all suffixes
- Per suffix produces: group block, total block, and exclusive block (members only in that format)
- Returns a single persisted unioned DataFrame across all blocks and suffixes

**Output Schema (all blocks share this structure):**

| Column | Description |
|---|---|
| `section` | Product format label |
| `segment` | Display name for the segment dimension |
| `cohort` | segment or overlap cohort |
| `feature` | Feature value or `All` for totals |
| `avg_sg` | Average sales growth (decimal, not %) |
| `total_mems` | Distinct member count |
| `pct_in_group` | Share of members within the cohort |
| `total_avg_sales_vals` | Count of members with non-null sales growth |
| `total_sales` | Sum of current window sales |
| `total_sales_prev` | Sum of previous window sales |
| `sales_growth` | Aggregate sales growth: `(total_sales - total_sales_prev) / total_sales_prev` |

**Notes:**
- `avg_sg` is the mean of member-level `sales_growth` values; `sales_growth` is computed from aggregated totals - these will differ and measure different things
- `multi_row_feature=True` must be set for any feature where a member can have multiple rows (e.g. `child_age_band`) - otherwise deduplication will incorrectly drop valid rows
- `feature = 'All'` is a placeholder used in total and overlap blocks where no feature breakdown is applied


## `04_x.ipynb`

**Description:** Computes benchmark-level feature profiles and summary statistics, with no segmentation or product format suffixes. Produces flat aggregation blocks that share the same output schema as the 03.1_comparisons.ipynb, allowing direct union and comparison downstream.

**Inputs:**

| Parameter | Source | Description |
|---|---|---|
| `x_df` | 01_data_no_segmentation notebook | Member-level sales data for the broader category of interest (no brand/article filter) |
| `shoprite_features` | `02_shoprite_features.ipynb` | Enriched member feature table |
| `section_label` | Caller-defined | Display label for the `section` column |
| `total_group_label` | Caller-defined | Display label for the `cohort` column on total rows |

> Requires `02_shoprite_features.ipynb` and the 01_data_no_segmentation.ipynb to have been run first.

**Key Functions:**

**`get_feature_total(feature_name, df, features_df, segment_name, section_label, total_group_label, multi_row_feature)`**
- Aggregates stats per feature value across the entire `df` with no `brand_segment` grouping and no suffix logic
- Joins `features_df` on `member_id`, filters nulls, then groups by `feature_name`
- Outputs one row per feature value with member counts, sales totals, avg sales growth, and aggregate sales growth
- `multi_row_feature=True` skips deduplication for features like `child_age_band` where a member legitimately has multiple rows

**`get_cohort_stats(df, section_label, cohort_label, segment_name)`**
- Simplest aggregation - no grouping, no feature join, no segment logic
- Collapses an entire DataFrame to a single summary row
- `section`, `cohort`, and `segment` are fully manual labels passed by the caller
- Used to produce top-level benchmark totals (e.g. total soap category stats)

**Output Schema (consistent with segmentation profiling notebook):**

| Column | Description |
|---|---|
| `section` | Caller-defined label (e.g. `Soap Category Benchmark`) |
| `segment` | Caller-defined label or `All` |
| `cohort` | Caller-defined label |
| `feature` | Feature value (e.g. `Female`) or `All` for cohort-level rows |
| `avg_sg` | Mean of member-level sales growth (decimal) |
| `total_mems` | Distinct member count |
| `pct_in_group` | Share of members within the group |
| `total_avg_sales_vals` | Count of members with non-null sales growth |
| `total_sales` | Sum of current window sales |
| `total_sales_prev` | Sum of previous window sales |
| `sales_growth` | Aggregate sales growth: `(total_sales - total_sales_prev) / total_sales_prev` |

**Notes:**
- All blocks produced in this notebook share an identical schema with the output of `03.1_comparisons.ipynb` - this is intentional so that all blocks can be unioned into a single reporting DataFrame at the end of the file
- `avg_sg` and `sales_growth` measure different things - `avg_sg` is the mean of per-member growth rates, `sales_growth` is derived from aggregated sales totals
- `common_args` is defined as a shared dict (`df=x_df`, `features_df=shoprite_features`, `section_label`, `total_group_label`) to reduce repetition across `get_feature_total` calls
- `multi_row_feature=True` must be explicitly set for any feature where one member can have multiple rows - omitting it will silently under-count members

# Post-Shoprite Run: Omnisient Upload & Project Setup

> Complete these steps **after** the Shoprite pipeline run has finished successfully.

---

## Step 1: Upload Output Datasets

Upload the following datasets to Omnisient.

### Benchmark Datasets
> Output of `01_data_no_segmentation.ipynb`

- `auto_capsules`
- `auto_liquid`
- `auto_powder`
- `auto_detergent`
- `soap_and_soap_powders`

### Segmented Customer Datasets
> Output of `01.3_data_segmentation.ipynb`

- `auto_capsules_segmented`
- `auto_liquid_segmented`
- `auto_powder_segmented`

---

## Step 2: Create Omnisient Project

Once all datasets are uploaded, create a new Omnisient project using the datasets below.

| Dataset | Description |
|---|---|
| `auto_capsules` | Benchmark |
| `auto_liquid` | Benchmark |
| `auto_powder` | Benchmark |
| `auto_detergent` | Benchmark |
| `soap_and_soap_powders` | Benchmark |
| `auto_capsules_segmented` | Segmented target group |
| `auto_liquid_segmented` | Segmented target group |
| `auto_powder_segmented` | Segmented target group |
| `STRIVE_CORE_SHARE` | STRIVE |
| `RM_Customer_Enhanced` | STRIVE |
| `VehicleData20260327` *(or most recent)* | STRIVE |
| `Shoprite_Combined_Dataset` *(All Shoprite & Checkers Members)* | Shoprite |

> **Note:** Always select the most recently dated vehicle dataset available (e.g. `VehicleData20260327` or newer).

# Omnisient Pipeline once Project is created.

## 01_data_prep.ipynb

Pulls segmented customer datasets from the Omnisient environment, standardises data types, performs a full outer join across all three product segments, computes overlap flags and sales growth per segment, and writes the final unified dataset back to the project SQL environment for downstream analysis.

---

### Requirements

#### Libraries

```python
numpy
pandas
matplotlib
seaborn
omni_hub     # Omnisient query extension
omni_lab     # Omnisient write utility
```

#### Input Datasets

The following tables must exist in the Omnisient project (`Shoprite_Prj_00393`) before running:

| Table | Description |
|---|---|
| `jurgensCapsulesSegmented_1` | Segmented customers - Auto Capsules |
| `jurgensLiquidSegmented_1` | Segmented customers - Auto Liquid |
| `jurgensPowderSegmented_1` | Segmented customers - Auto Powder |

Each table is expected to share the same schema, including the following columns:

| Column | Type |
|---|---|
| `member_id_SKey` | Identifier (unique per member) |
| `brand_segment` | Segment label |
| `sales_value`, `total_items`, `avg_sales_value` | Current period metrics |
| `prev_sales_value`, `prev_total_items`, `prev_avg_sales_value` | Prior period metrics |
| `max_date_mem`, `prev_max_date_mem` | Date columns |

---

### Steps

#### 1. Load Data
Queries all three segmented tables from `Shoprite_Prj_00393` via `%omni_hub` and loads them into DataFrames.

#### 2. Cast Data Types
Applies consistent dtype casting across all DataFrames:
- Numeric columns → `float64` via `pd.to_numeric`
- Date columns → `datetime64` via `pd.to_datetime`

#### 3. Join Datasets
Performs a full outer join across all three DataFrames on `member_id_SKey`. For each product segment, the function:
- Deduplicates on `member_id_SKey`
- Renames columns with a product suffix (`_liq`, `_powd`, `_caps`)
- Computes **sales growth** `((current - previous) / previous) * 100`
- Computes **overlap flags** per segment:

| Flag | Meaning |
|---|---|
| `in_exclusive_{segment}` | Member appears in this segment only |
| `in_one_other_{segment}` | Member appears in this segment + one other |
| `in_all_{segment}` | Member appears in all three segments |

#### 4. Write to SQL Environment
Uploads the final joined DataFrame (~x rows × 34 columns) to the project SQL environment:

```
Shoprite_Prj_00393.df_final_join
```

---

### Output

| Table | Rows | Columns |
|---|---|---|
| `df_final_join` | ~870,902 | 34 |

---

### Next Step - SQLPad Join

`df_final_join` must be joined with the Shoprite member base and any additional datasets required for analysis. This is done in the SQLPad environment. Example:

```sql
CREATE TABLE strive_omo_df
WITH (DISTRIBUTION = ROUND_ROBIN, HEAP) AS
SELECT a.*, c.*
FROM Shoprite_Prj_00393.df_final_join AS a
INNER JOIN Shoprite_Prj_00393.Shoprite_Combined_Dataset_10_MATCHING AS b
    ON a.member_id_SKey = b.MEMBERKEY_SKEY
INNER JOIN Shoprite_Prj_00393.STRIVE_CORE_SHARE_11 AS c
    ON b.DW_ID_KEY = c.DW_ID_Key
```

## `02_strive_features.ipynb`

Loads the joined Strive + Shoprite dataset, computes demographic and socioeconomic feature aggregates across all product segments and brand cohorts, runs the same aggregations across all benchmark product groups, and writes a unified aggregate output back to the project SQL environment.

---

### Requirements

#### Libraries

```python
numpy
pandas
matplotlib
seaborn
omni_hub     # Omnisient query extension
omni_lab     # Omnisient write utility
```

#### Input Datasets

The following tables must exist in `Shoprite_Prj_00393` before running. The segmented input is produced by `01_data_prep.ipynb` + the SQLPad join step. The benchmark inputs require their own SQLPad joins prior to running this notebook.

| Table | Description |
|---|---|
| `strive_omo_df` | Joined segmented dataset (output of `01_data_prep` → SQLPad) |
| `strive_capsule_benchmark_df` | Benchmark - Auto Capsules joined with Strive |
| `strive_auto_detergent_benchmark_df` | Benchmark - Auto Detergent joined with Strive |
| `strive_liquid_benchmark_df` | Benchmark - Auto Liquid joined with Strive |
| `strive_powder_benchmark_df` | Benchmark - Auto Powder joined with Strive |
| `strive_soap_benchmark_df` | Benchmark - Soap & Soap Powders joined with Strive |

---

### Steps

#### 1. Load Data
Queries `strive_omo_df` from `Shoprite_Prj_00393` via `%omni_hub` and loads it into a DataFrame.

#### 2. Cast Data Types
Applies consistent dtype casting:
- Numeric columns → `float64` via `pd.to_numeric`
- Date columns → `datetime64` via `pd.to_datetime`

#### 3. Build Aggregate Functions
Three functions handle feature profiling:

**`_build_group_blocks`** - Produces two output blocks per product suffix:
- A breakdown by `brand_segment` × feature value
- A total rollup across the entire group

**`_build_overlap_block`** - Produces a feature breakdown filtered to a specific overlap cohort (`exclusive`, `one_other`, or `in_all`).

**`get_feature_data_a`** - Orchestrates the above for each product suffix (`liq`, `powd`, `caps`), producing five blocks per suffix: group, total, exclusive, one\_other, and in\_all.

**`get_feature_total`** - Used for benchmark datasets. Calculates total stats per feature value with no brand\_segment grouping and no suffix logic.

Each output block contains the following columns:

| Column | Description |
|---|---|
| `section` | Product group label |
| `segment` | Feature display name |
| `cohort` | Brand segment or overlap cohort |
| `feature` | Feature value |
| `avg_sg` | Average sales growth (decimal) |
| `total_mems` | Unique member count |
| `pct_in_group` | Share of members within cohort |
| `total_avg_sales_vals` | Count of members with valid sales growth |
| `total_sales` | Total current period sales |
| `total_sales_prev` | Total prior period sales |
| `sales_growth` | Period-over-period sales growth |

#### 4. Run Across STRIVE Features
The following 13 STRIVE features are profiled across all three segmented product groups:

`Age_Group`, `Marital_Status_YN`, `Director_YN`, `Entrepreneur_YN`, `Homeowner_YN`, `Number_Of_Homes`, `Credit_Active_yn`, `Salary_Prediction_4Groups`, `Social_Class`, `Province`, `Vehicleowner_YN`, `Number_Of_Vehicles`, `LSM`

Output → `strive_segmented_df`

#### 5. Load & Aggregate Benchmark Datasets
Loads each benchmark table, casts dtypes, and runs `get_feature_total` across the same 13 features for each product group:

| Input Table | Output DataFrame |
|---|---|
| `strive_capsule_benchmark_df` | `strive_caps_final` |
| `strive_auto_detergent_benchmark_df` | `strive_detergent_final` |
| `strive_liquid_benchmark_df` | `strive_liq_final` |
| `strive_powder_benchmark_df` | `strive_powd_final` |
| `strive_soap_benchmark_df` | `strive_soap_final` |

All five are concatenated → `benchmark_df`

#### 6. Combine & Write
`strive_segmented_df` and `benchmark_df` are concatenated into `final_aggregate_strive` and uploaded to the project SQL environment:

```
Shoprite_Prj_00393.strive_aggregates_df
```

---

## `03_rm_features.ipynb`

Loads the joined RM Customer + Shoprite dataset, computes retail credit score and basket score aggregates across all product segments and brand cohorts, runs the same aggregations across all benchmark product groups, and writes a unified aggregate output back to the project SQL environment.

---

### Requirements

#### Libraries

```python
numpy
pandas
matplotlib
seaborn
omni_hub     # Omnisient query extension
omni_lab     # Omnisient write utility
```

#### Input Datasets

The following tables must exist in `Shoprite_Prj_00393` before running. The segmented input is produced by `01_data_prep.ipynb` + the SQLPad join step. The benchmark inputs require their own SQLPad joins with `RM_Customer_Enhanced` prior to running this notebook.

| Table | Description |
|---|---|
| `rm_omo_df` | Joined segmented dataset (output of `01_data_prep` → SQLPad join with `RM_Customer_Enhanced`) |
| `rm_capsule_benchmark_df` | Benchmark — Auto Capsules joined with RM Customer |
| `rm_auto_detergent_benchmark_df` | Benchmark — Auto Detergent joined with RM Customer |
| `rm_liquid_benchmark_df` | Benchmark — Auto Liquid joined with RM Customer |
| `rm_powder_benchmark_df` | Benchmark — Auto Powder joined with RM Customer |
| `rm_soap_benchmark_df` | Benchmark — Soap & Soap Powders joined with RM Customer |

> **SQLPad join example** (benchmark tables):
> ```sql
> CREATE TABLE rm_capsule_benchmark_df
> WITH (DISTRIBUTION = ROUND_ROBIN, HEAP) AS
> SELECT a.*, b.SRX_Retail_Credit_Score, b.SRX_Retail_Credit_Score_Category,
>        b.BasketScore_PL_Category, b.BasketScore_PL
> FROM Shoprite_Prj_00393.jurgensCapsulesBenchmark_1 AS a
> INNER JOIN Shoprite_Prj_00393.RM_Customer_Enhanced_1 AS b
>     ON a.member_id_SKey = b.MEMBERKEY_SKey
> ```

---

### Steps

#### 1. Load Data
Queries `rm_omo_df` from `Shoprite_Prj_00393` via `%omni_hub` and loads it into a DataFrame.

#### 2. Cast Data Types
Applies consistent dtype casting:
- Numeric columns → `float64` via `pd.to_numeric`
- Date columns → `datetime64` via `pd.to_datetime`

#### 3. Build Aggregate Functions
Three functions handle feature profiling:

**`_build_group_blocks`** - Produces two output blocks per product suffix:
- A breakdown by `brand_segment` × feature value
- A total rollup across the entire group

**`_build_overlap_block`** - Produces a feature breakdown filtered to a specific overlap cohort (`exclusive`, `one_other`, or `in_all`).

**`get_feature_data_a`** - Orchestrates the above for each product suffix (`liq`, `powd`, `caps`), producing five blocks per suffix: group, total, exclusive, one\_other, and in\_all.

**`get_feature_total`** - Used for benchmark datasets. Calculates total stats per feature value with no brand\_segment grouping and no suffix logic.

Each output block contains the following columns:

| Column | Description |
|---|---|
| `section` | Product group label |
| `segment` | Feature display name |
| `cohort` | Brand segment or overlap cohort |
| `feature` | Feature value |
| `avg_sg` | Average sales growth (decimal) |
| `total_mems` | Unique member count |
| `pct_in_group` | Share of members within cohort |
| `total_avg_sales_vals` | Count of members with valid sales growth |
| `total_sales` | Total current period sales |
| `total_sales_prev` | Total prior period sales |
| `sales_growth` | Period-over-period sales growth |

#### 4. Run Across RM Features
The following 2 RM Customer features are profiled across all three segmented product groups:

`SRX_Retail_Credit_Score_Category`, `BasketScore_PL_Category`

Output → `rm_segmented_df`

#### 5. Load & Aggregate Benchmark Datasets
Loads each benchmark table, casts dtypes, and runs `get_feature_total` across the same 2 features for each product group:

| Input Table | Output DataFrame |
|---|---|
| `rm_capsule_benchmark_df` | `rm_caps_final` |
| `rm_auto_detergent_benchmark_df` | `rm_detergent_final` |
| `rm_liquid_benchmark_df` | `rm_liq_final` |
| `rm_powder_benchmark_df` | `rm_powd_final` |
| `rm_soap_benchmark_df` | `rm_soap_final` |

All five are concatenated → `benchmark_df`

#### 6. Combine & Write
`rm_segmented_df` and `benchmark_df` are concatenated into `final_aggregate_rm` and uploaded to the project SQL environment:

```
Shoprite_Prj_00393.rm_aggregates_df
```

---


## `04_vehicle_features.ipynb`

Loads the joined Vehicle + Shoprite dataset, computes vehicle make and vehicle cohort aggregates across all product segments and brand cohorts, runs the same aggregations across all benchmark product groups, and writes a unified aggregate output back to the project SQL environment.

---

### Requirements

#### Libraries

```python
numpy
pandas
matplotlib
seaborn
omni_hub     # Omnisient query extension
omni_lab     # Omnisient write utility
```

#### Input Datasets

The following tables must exist in `Shoprite_Prj_00393` before running. The segmented input requires a SQLPad join with `VehicleData` prior to running this notebook. The benchmark inputs require their own equivalent SQLPad joins.

| Table | Description |
|---|---|
| `vehicle_omo_df` | Joined segmented dataset (output of `01_data_prep` → SQLPad join with `VehicleData`) |
| `vehicle_capsule_benchmark_df` | Benchmark - Auto Capsules joined with Vehicle data |
| `vehicle_auto_detergent_benchmark_df` | Benchmark - Auto Detergent joined with Vehicle data |
| `vehicle_liquid_benchmark_df` | Benchmark - Auto Liquid joined with Vehicle data |
| `vehicle_powder_benchmark_df` | Benchmark - Auto Powder joined with Vehicle data |
| `vehicle_soap_benchmark_df` | Benchmark - Soap & Soap Powders joined with Vehicle data |

> **SQLPad join example** (segmented input table):
> ```sql
> CREATE TABLE vehicle_omo_df
> WITH (DISTRIBUTION = ROUND_ROBIN, HEAP) AS
> SELECT a.*, c.DW_ID_Key, c.Vehicle_Make, c.Vehicle_Model, c.Vehicle_Year, c.Vehicle_Owner,
>     CASE
>         WHEN c.Strive_Cohorts_Fleet_Owner      = 'Y' THEN 'Fleet_Owner'
>         WHEN c.Strive_Cohorts_Luxury_Owner     = 'Y' THEN 'Luxury_Owner'
>         WHEN c.Strive_Cohorts_French_Flair     = 'Y' THEN 'French_Flair'
>         WHEN c.Strive_Cohorts_Classic_Collector = 'Y' THEN 'Classic_Collector'
>         WHEN c.Strive_Cohorts_German_Luxury    = 'Y' THEN 'German_Luxury'
>         WHEN c.Strive_Cohorts_Bakkie_Brigade   = 'Y' THEN 'Bakkie_Brigade'
>         WHEN c.Strive_Cohorts_Taxi_Owner       = 'Y' THEN 'Taxi_Owner'
>         WHEN c.Strive_Cohorts_Petrolhead       = 'Y' THEN 'Petrolhead'
>         WHEN c.Strive_Cohorts_Parking_Lot_Mom  = 'Y' THEN 'Parking_Lot_Mom'
>         WHEN c.Strive_Cohorts_Rust_Buckets     = 'Y' THEN 'Rust_Buckets'
>         WHEN c.Strive_Cohorts_Twenty_Plenty    = 'Y' THEN 'Twenty_Plenty'
>         WHEN c.Strive_Cohorts_4X4_Lovers       = 'Y' THEN '4X4_Lovers'
>         WHEN c.Strive_Cohorts_Truck_Owner      = 'Y' THEN 'Truck_Owner'
>         WHEN c.Strive_Cohorts_Bike_Club        = 'Y' THEN 'Bike_Club'
>         WHEN c.Strive_Cohorts_Farmer           = 'Y' THEN 'Farmer'
>         WHEN c.Strive_Cohorts_Asian_Newcomers  = 'Y' THEN 'Asian_Newcomers'
>         WHEN c.Strive_Cohorts_Electrical_Vehicle = 'Y' THEN 'Electrical_Vehicle'
>         ELSE ''
>     END AS vehicle_cohorts
> FROM Shoprite_Prj_00393.df_final_join AS a
> INNER JOIN Shoprite_Prj_00393.Shoprite_Combined_Dataset_10_MATCHING AS b
>     ON a.member_id_SKey = b.MEMBERKEY_SKEY
> INNER JOIN Shoprite_Prj_00393.VehicleData20260327_12 AS c
>     ON b.DW_ID_KEY = c.DW_ID_Key
> ```

---

### Steps

#### 1. Load Data
Queries `vehicle_omo_df` from `Shoprite_Prj_00393` via `%omni_hub` and loads it into a DataFrame.

#### 2. Cast Data Types
Applies consistent dtype casting:
- Numeric columns → `float64` via `pd.to_numeric`
- Date columns → `datetime64` via `pd.to_datetime`

#### 3. Build Aggregate Functions
Three functions handle feature profiling:

**`_build_group_blocks`** - Produces two output blocks per product suffix:
- A breakdown by `brand_segment` × feature value
- A total rollup across the entire group

Supports an optional `include_values` filter to restrict output to a specific list of feature values (e.g. `["Toyota", "Ford", "BMW"]` for `Vehicle_Make`).

**`_build_overlap_block`** - Produces a feature breakdown filtered to a specific overlap cohort (`exclusive`, `one_other`, or `in_all`). Also supports `include_values` filtering.

**`get_feature_data_a`** - Orchestrates the above for each product suffix (`liq`, `powd`, `caps`), producing group, total, exclusive, one\_other, and in\_all blocks per suffix.

**`get_feature_total`** - Used for benchmark datasets. Calculates total stats per feature value with no brand\_segment grouping and no suffix logic. Also supports `include_values` filtering.

Each output block contains the following columns:

| Column | Description |
|---|---|
| `section` | Product group label |
| `segment` | Feature display name |
| `cohort` | Brand segment or overlap cohort |
| `feature` | Feature value |
| `avg_sg` | Average sales growth (decimal) |
| `total_mems` | Unique member count |
| `pct_in_group` | Share of members within cohort |
| `total_avg_sales_vals` | Count of members with valid sales growth |
| `total_sales` | Total current period sales |
| `total_sales_prev` | Total prior period sales |
| `sales_growth` | Period-over-period sales growth |

#### 4. Run Across Vehicle Features
The following 2 vehicle features are profiled across all three segmented product groups:

| Feature | Description | Filter Applied |
|---|---|---|
| `Vehicle_Make` | Manufacturer of the vehicle | Toyota, Ford, BMW only |
| `vehicle_cohorts` | Derived vehicle lifestyle cohort | All cohorts |

Output → `vehicle_segmented_df`

#### 5. Load & Aggregate Benchmark Datasets
Loads each benchmark table, casts dtypes, and runs `get_feature_total` across the same 2 features for each product group:

| Input Table | Output DataFrame |
|---|---|
| `vehicle_capsule_benchmark_df` | `vehicle_caps_final` |
| `vehicle_auto_detergent_benchmark_df` | `vehicle_detergent_final` |
| `vehicle_liquid_benchmark_df` | `vehicle_liq_final` |
| `vehicle_powder_benchmark_df` | `vehicle_powd_final` |
| `vehicle_soap_benchmark_df` | `vehicle_soap_final` |

All five are concatenated → `benchmark_df`

#### 6. Combine & Write
`vehicle_segmented_df` and `benchmark_df` are concatenated into `final_aggregate_vehicle` and uploaded to the project SQL environment:

```
Shoprite_Prj_00393.vehicle_aggregates_df
```

---

# All Aggregate features are now created:

- Aggregates from `02_strive_features.ipynb`: 'vehicle_aggregates_df'
- Aggregates from `03_rm_features.ipynb`: 'rm_aggregates_df'
- Aggregates from `04_vehicle_features.ipynb`: 'strive_aggregates_df'
- Aggregates from `03.1_comparisons.ipynb`:  'customer_segments_aggregates'
- Aggregates from `04_x.ipynb`: benchmark aggregates: 'capsules', 'liquid', 'powder', 'auto detergent' and 'soap and soap powders'

With the aggregates above we can create the Tableau Dashboards. This changes based on number of comparison groups we are inspecting, the indexes of interest, and the depth of Tableau analysis required for a given product, category etc.
