# Data Analyst Job Market Analysis (SQL Project)

## Introduction

This project explores the job market for **Data Analyst** roles using SQL, digging into which positions pay the most, which skills those top-paying jobs require, which skills are in the highest overall demand, and — most importantly — which skills sit at the sweet spot of being both **in-demand** and **well-paid**. The goal was to answer a practical question: *if I'm learning skills for a Data Analyst career, where should I focus my time?*

Each SQL file in this repo answers one specific question and is documented with a comment block at the top explaining the query and, in some cases, the resulting output and a short takeaway.

## Background

Data driven by real job posting data, this project was built to simulate the kind of exploratory analysis a Data Analyst might be asked to do on their own career: mining a large postings dataset to surface trends in compensation and skill demand. The dataset includes job postings with fields such as job title, location, salary, company, and posting date, joined against a normalized skills schema (`skills_dim`, `skills_job_dim`) that maps each posting to the technologies/tools it requires.

### The questions this project answers:

1. What are the top-paying remote Data Analyst jobs?
2. What skills are required for those top-paying remote jobs?
3. What are the most in-demand skills for Data Analysts overall?
4. Which skills are associated with the highest average salaries?
5. What are the most *optimal* skills to learn (high demand **and** high pay)?

## Tools I Used

- **SQL (PostgreSQL)** — the core language for querying and analyzing the dataset
- **PostgreSQL** — database engine used to host and query the job postings dataset
- **Visual Studio Code** — for writing and managing SQL scripts
- **Git & GitHub** — version control and project sharing

## The Analysis

Each query targets a specific angle of the job market. Here's a breakdown of the approach for each:

### 1. Top Paying Data Analyst Jobs (`top_paying_jobs.sql`)

Filtered for remote (`job_location = 'Anywhere'`) Data Analyst roles with a listed salary, then sorted by `salary_year_avg` to surface the top 10 highest-paying postings.

```sql
SELECT
    job_id,
    job_title,
    job_location,
    job_schedule_type,
    salary_year_avg,
    job_posted_date,
    name as company_name
FROM 
    job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE 
    salary_year_avg IS NOT NULL AND 
    job_title_short = 'Data Analyst' AND 
    job_location = 'Anywhere'
ORDER BY 
    salary_year_avg DESC
LIMIT 10;
```

**Key finding:** The top-paying remote Data Analyst posting was a role at **Mantys** listed at **$650,000/year** — a significant outlier compared to the rest of the field. Excluding that top result, salaries settle into a more typical (though still high) $184K–$336K range, led by **Director of Analytics at Meta** ($336,500) and **Associate Director – Data Insights at AT&T** ($255,829.50). Notably, several of the highest-paying "Data Analyst" postings are actually senior/director-level titles (Director, Principal, Associate Director), suggesting the biggest pay jumps in this field come from seniority and scope rather than the base "Data Analyst" title alone.

### 2. Skills for Top Paying Jobs (`top_paying_job_skills.sql`)

Took the top 10 jobs from the query above (as a CTE) and joined them against the skills tables to see exactly what each of those roles required.

**Key finding:** SQL appeared in every single one of the top 10 highest-paying remote postings (100%), followed by Python (~87.5%) and Tableau (~75%). High-paying Data Analyst roles consistently combine core analytical skills (SQL) with programming (Python/R) and BI/visualization tools (Tableau, Power BI), and several also lean on cloud platforms (Azure, AWS, Snowflake).

### 3. Top Demanded Skills (`top_demanded_skills.sql`)

Looked across *all* Data Analyst postings (not just top-paying ones) and counted how often each skill appeared, to find the top 5 most requested skills overall.

```sql
SELECT
    skills,
    COUNT(job_postings_fact.job_id) AS demand_count
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
GROUP BY
    skills
ORDER BY
    demand_count DESC
LIMIT 5;
```

**Key finding:** The top 5 most demanded skills across all Data Analyst postings are **SQL** (92,628 mentions), **Excel** (67,031), **Python** (57,326), **Tableau** (46,554), and **Power BI** (39,468). SQL and Excel remain the clear foundation of the role by a wide margin, while Python's strong showing signals that programming ability is now a mainstream (not niche) expectation, and the two BI tools confirm that visualization/reporting skills are just as essential as the analytical ones.

### 4. Top Paying Skills (`top_paying_skills.sql`)

Grouped by skill and averaged the salary of every posting that listed it, to find which individual skills are associated with the highest pay — regardless of how common they are.

```sql
SELECT
    skills,
    ROUND(AVG(salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst' AND
    salary_year_avg IS NOT NULL
GROUP BY
    skills
ORDER BY
    avg_salary DESC
LIMIT 25;
```

**Key finding:** The highest-paying individual skill by a huge margin is **SVN** at an average of **$400,000** — almost certainly a single outlier posting skewing the average rather than a broad market trend, given how rarely SVN (an older version control system) appears in modern Data Analyst roles. Excluding that outlier, the top of the list is dominated by specialized/emerging tech: **Solidity** ($179,000), **Couchbase** ($160,515), **DataRobot** ($155,486), and **Golang** ($155,000), followed by a cluster of ML/data-engineering tools like **PyTorch**, **TensorFlow**, **Keras**, **Kafka**, and **Airflow** all averaging $115K–$130K. This paints a clear picture: the highest *average* salaries go to niche, specialized, or ML/engineering-adjacent skills — but as the note below shows, high pay alone doesn't mean high opportunity, since many of these skills appear in very few postings.

### 5. Most Optimal Skills (`optimal_skills.sql`)

Combined demand and salary into a single query, filtering out low-signal/niche skills (`HAVING COUNT(...) > 10`) so the result only surfaces skills that are both genuinely in demand *and* well paid — the ones worth prioritizing when learning.

```sql
SELECT
    skills_dim.skill_id,
    skills_dim.skills,
    COUNT(job_postings_fact.job_id) AS demand_count,
    ROUND(AVG(job_postings_fact.salary_year_avg)) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst' AND
    salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY
    skills_dim.skill_id
HAVING
    COUNT(job_postings_fact.job_id) > 10
ORDER BY
    avg_salary DESC,
    demand_count DESC;
```

**Key finding:** Once low-demand outliers are filtered out (`> 10` postings), the picture looks very different from the raw top-paying-skills list above. **Go** leads at $115,320 (27 postings), followed by **Confluence** ($114,210), **Hadoop** ($113,193), **Snowflake** ($112,948, 37 postings), and **Azure** ($111,225, 34 postings). Further down, the highly *demanded* core skills reappear with strong, consistent pay: **Python** ($101,397 across 236 postings), **R** ($100,499 across 148 postings), **Tableau** ($99,288 across 230 postings), **Power BI** ($97,431 across 110 postings), and **SQL** ($97,237 across 398 postings — the single most demanded skill in the entire dataset). This is the most actionable list in the project: it confirms that SQL, Python, R, Tableau, and Power BI aren't just common — they're reliably well-paid too, while cloud platform skills (Snowflake, Azure, AWS) offer a meaningful salary bump for those willing to specialize further.

## What I Learned

- **CTEs make layered questions readable.** Structuring `top_paying_job_skills.sql` as a `WITH` clause first, then joining skills onto it, kept the query easy to follow instead of nesting subqueries.
- **A single outlier can distort an average.** Both the top-paying-jobs query ($650,000 Mantys posting) and the top-paying-skills query (SVN at $400,000) had one extreme value pulling attention away from the more representative numbers around them. It's a good reminder to sanity-check `AVG()` results against `COUNT()` before drawing conclusions.
- **Demand and pay don't always line up.** Comparing the results side by side made this concrete: SQL is the #1 most demanded skill (92,628 postings) but ranks 27th in the pure top-paying-skills list ($97,237) — meanwhile niche tools like Solidity or DataRobot pay more on average but appear in far fewer postings. The "optimal skills" query, which combines both signals with a `HAVING COUNT(...) > 10` filter, was the only one that surfaced genuinely actionable skills rather than one-off high-salary flukes.
- **`LEFT JOIN` vs `INNER JOIN` matters for completeness.** Using `LEFT JOIN` against `company_dim` in the top-paying-jobs query ensures postings aren't dropped just because a company record is missing, while `INNER JOIN` was the right call for the skills tables since a job with no matched skills isn't useful for a skills-focused analysis.
- **Filtering before aggregating keeps results meaningful.** The `HAVING COUNT(...) > 10` clause in `optimal_skills.sql` is what separates it from the noisier `top_paying_skills.sql` results — without it, a skill like SVN with a single high-salary posting can rank above genuinely valuable, widely-used skills.
- **Small syntax slips are easy to miss.** Caught a `GrOUP BY` typo (mixed casing) while reviewing `optimal_skills.sql` — SQL is case-insensitive for keywords so it still ran, but it's a good reminder to double check formatting consistency across files.

## Conclusion

This analysis paints a clear picture of what it takes to land a well-paying Data Analyst role, especially remote ones. **SQL is non-negotiable** — it's both the most demanded skill in the market (92,628 postings) and appeared in 100% of the top 10 highest-paying remote postings analyzed. Beyond that, **Python, R, Tableau, and Power BI** show up consistently across both the demand and optimal-skills results, confirming they're not just common but reliably well-compensated — and familiarity with **cloud/data platforms** (Snowflake, Azure, AWS) offers a real salary premium on top of that foundation.

The raw "top paying skills" list is tempting but misleading on its own — high averages like SVN's $400,000 or Solidity's $179,000 are driven by a small number of postings, not broad market demand. The "optimal skills" query is the most actionable result in this project: it shows that the smartest skills to prioritize aren't the rarest, highest-paying ones, but the ones that combine **strong demand with strong pay** — which, in this dataset, means SQL, Python, R, Tableau, Power BI, and select cloud platforms.