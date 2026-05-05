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


## 2. `02_member_base_profiling.ipynb`

**Description:** Constructs a desired member base from Athena and generates a profiled feature summary across demographic and behavioural segments for Checkers customers.

**Inputs:**
| Parameter | Description |
|---|---|
| `end_date` | End date for the analysis window |
| `look_back_interval` | Number of weeks to look back from end date |
| `banner` | Banner filter applied to transaction data (e.g. `CHECKERS`) |
| `country_code` | Country filter for transaction data (e.g. `ZA`) |
| `member_base` | Spark DataFrame of members used as the base population |
| `member_feature_base_df` | Spark DataFrame containing feature columns joined to members |
| `section_label` | Label for the reporting section |
| `total_group_label` | Label for the cohort total row |

**Outputs:**
- A unified Spark DataFrame (`df_all_features`) containing member counts and percentage breakdowns across all feature segments, including demographics (generation, gender, affluence, parental status, child age), pet ownership (type and counts).
- Each row is keyed by `section`, `segment`, `cohort`, and `feature` for downstream reporting use.

**Notes:**
- The Athena query filters out department `99`, zero/negative sales, and null ticket times to ensure clean transaction data.
- `get_feature_total` supports both single-row and multi-row features via the `multi_row_feature` flag - use `True` for features where a member can have multiple rows (e.g. child ages).
- Sales metric columns (`total_avg_sales_vals`, `total_sales`, `total_sales_prev`, `sales_growth`) are `null` as they are not needed for this set.
- `df_shoprite_shabit` is currently commented out and excluded from the union: Since we are inspecting checkers base.

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







