# Introduction
This project uses SQL to look at the data analyst job market. I wanted to find out which roles pay the most, which skills employers ask for most often, and which skills are worth learning if you want both demand and a good salary.

The SQL queries are in the [project_sql folder](/project_sql/).

# Background
I'm learning SQL alongside my job as a Commercial Analyst, and I wanted a project that used real data to answer real questions, rather than just practising syntax. This one helped me work out what to focus on next.

The data comes from [Luke Barousse's SQL course](https://lukebarousse.com/sql). It has job postings with titles, salaries, locations and skills.

### The questions I wanted to answer

1. What are the top-paying data analyst jobs?
2. What skills do those top-paying jobs require?
3. What skills are most in demand for data analysts?
4. Which skills are linked to higher salaries?
5. What are the most optimal skills to learn?

# Tools I Used
- **SQL:** for all the analysis
- **PostgreSQL:** the database the job postings sit in
- **Visual Studio Code:** for writing and running the queries
- **Git & GitHub:** for version control and sharing the project

# The Analysis
Each query answers one of the questions above. Here's what I did for each.

### 1a. Top Paying Data Analyst Jobs (Remote)
I filtered for remote data analyst jobs that list a yearly salary, then sorted by salary to see the top 10.

```sql
SELECT
    job_id,
    job_title,
    job_location,
    job_schedule_type,
    salary_year_avg,
    job_posted_date,
    name AS company_name
FROM
    job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE
    job_title_short = 'Data Analyst' AND
    job_location = 'Anywhere' AND
    salary_year_avg IS NOT NULL
ORDER BY
    salary_year_avg DESC
LIMIT 10;
```

| Job Title | Company | Average Yearly Salary ($) |
|-----------|---------|--------------------------:|
| Data Analyst | Mantys | 650,000 |
| Director of Analytics | Meta | 336,500 |
| Associate Director - Data Insights | AT&T | 255,830 |
| Data Analyst, Marketing | Pinterest Job Advertisements | 232,423 |
| Data Analyst (Hybrid/Remote) | Uclahealthcareers | 217,000 |
| Principal Data Analyst (Remote) | SmartAsset | 205,000 |
| Director, Data Analyst - HYBRID | Inclusively | 189,309 |
| Principal Data Analyst, AV Performance Analysis | Motional | 189,000 |
| Principal Data Analyst | SmartAsset | 186,000 |
| ERM Data Analyst | Get It Recruit - Information Technology | 184,000 |

*Top 10 highest-paying remote data analyst jobs with a listed salary*

**What I noticed:**
- The top 10 range from $184,000 to $650,000. The $650,000 Data Analyst role at Mantys is nearly double the next one ($336,500), so it looks like an outlier.
- Six of the ten are senior titles (Director, Associate Director or Principal), so the highest salaries mostly go with seniority rather than the plain "Data Analyst" title.
- The employers vary a lot, from Meta, AT&T and Pinterest to a healthcare employer, with SmartAsset appearing twice.
- All ten are full-time and were posted in 2023.

### 1b. Top Paying Data Analyst Jobs (UK)
I also wanted to see what the top-paying roles look like in the UK. My first attempt filtered `job_location` with `TRIM(job_location) ILIKE '%, UK'` (the column has trailing spaces), but that missed any job whose location is just "United Kingdom" with no city. The `search_location` column holds the country each job was searched under, so filtering on that is simpler and catches them.

```sql
SELECT
    job_id,
    job_title,
    job_location,
    job_schedule_type,
    salary_year_avg,
    job_posted_date,
    search_location,
    name AS company_name
FROM
    job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE
    job_title_short = 'Data Analyst' AND
    search_location = 'United Kingdom' AND
    salary_year_avg IS NOT NULL
ORDER BY
    salary_year_avg DESC
LIMIT 10;
```

| Job Title | Company | Location | Average Yearly Salary ($) |
|-----------|---------|----------|--------------------------:|
| Market Data Lead Analyst | Deutsche Bank | United Kingdom | 180,000 |
| Research Engineer, Science | DeepMind | London | 177,283 |
| Data Architect | AND Digital | Bristol | 165,000 |
| Data Analyst | Plexus Resource Solutions | Anywhere | 165,000 |
| Data Architect | Darktrace | Cambridge | 165,000 |
| Data Architect | Logispin | London | 163,782 |
| Data Architect - Trading and Supply | Shell | United Kingdom | 156,500 |
| Research Scientist, Science | DeepMind | London | 149,653 |
| Analytics Engineer - ENA London, Warsaw- (F/M) | AccorCorpo | London | 139,216 |
| Finance Data Analytics Manager | AJ Bell | Manchester | 132,500 |

*Top 10 highest-paying jobs under the Data Analyst category searched in the UK, with a listed salary*

**What I noticed:**
- The top 10 range from $132,500 to $180,000, well below the remote top 10 above.
- Only one role has a plain "Data Analyst" title, and its location is "Anywhere", so it's probably a remote job that was searched under the UK. The rest are things like Data Architect (four of the ten), Research Engineer and Analytics Engineer, so the "Data Analyst" category clearly includes other roles.
- London accounts for four of the ten. The others are spread across Bristol, Cambridge and Manchester, with three listed only as "United Kingdom" or "Anywhere".
- Using `search_location` found Deutsche Bank and Shell, which my first query missed. It was a good reminder to check how a filter treats the values in a column.

### 2. Skills for Top Paying Jobs
For this one I took the highest-paying data analyst jobs from my first query and looked at which skills they list. The idea is to see what's worth learning if you're aiming for the better-paid roles.

#### The query
I used a CTE to grab the top 10 highest-paying remote data analyst jobs, then joined that to the skills tables to get the skill names for each job.

The CTE (`top_paying_jobs`) pulls the job id, title, average yearly salary and company name. It only looks at:

- jobs with the title `Data Analyst`
- remote jobs (`job_location = 'Anywhere'`)
- jobs that actually list a salary

It's ordered by salary, highest first, and limited to 10. After that I joined it to `skills_job_dim` and `skills_dim` so each job comes back with one row per skill.

```sql
WITH top_paying_jobs AS (
    SELECT
        job_id,
        job_title,
        salary_year_avg,
        name AS company_name
    FROM job_postings_fact
    LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
    WHERE
        job_title_short = 'Data Analyst' AND
        job_location = 'Anywhere' AND
        salary_year_avg IS NOT NULL
    ORDER BY salary_year_avg DESC
    LIMIT 10
)

SELECT
    top_paying_jobs.*,
    skills
FROM top_paying_jobs
INNER JOIN skills_job_dim ON top_paying_jobs.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
ORDER BY salary_year_avg DESC;
```

#### Analysing the results
I exported the results as a CSV, uploaded it to Claude and asked it to go through the skills column and show me what stood out. Here's what it came back with:

<img width="814" height="828" alt="image" src="https://github.com/user-attachments/assets/bbb27400-a466-4238-85a3-582e459927a6" />

Interestingly, Power BI didn't appear as often as I expected, with Tableau the leading BI tool (6 of 8 postings vs 2). This is only a small sample of remote, top-paying roles, and the dataset leans towards US postings, so I wouldn't read too much into it. Still, it's made me think about focusing on Tableau next, and I'll check a wider set of data analyst postings before committing.

#### Why I only got 8 jobs, not 10
I put `LIMIT 10` in the query but the CSV only had 8 jobs in it, which confused me at first.

The reason is the order things happen in. The CTE picks the top 10 by salary without caring whether a job has any skills listed. Then the `INNER JOIN` to `skills_job_dim` throws away any job with no skills rows. Two of my top 10 had no skills listed, so they got dropped and I was left with 8.

To fix it, I need to check for skills inside the CTE, so the `LIMIT` only counts jobs that have them. I did this by adding one line to the `WHERE`, which only keeps jobs whose `job_id` appears in the skills table:

```sql
WITH top_paying_jobs AS (

SELECT
    job_id,
    job_title,
    salary_year_avg,
    name AS company_name
FROM
    job_postings_fact
LEFT JOIN company_dim ON job_postings_fact.company_id = company_dim.company_id
WHERE
    job_title_short = 'Data Analyst' AND
    job_location = 'Anywhere' AND
    salary_year_avg IS NOT NULL AND
    job_postings_fact.job_id IN (SELECT job_id FROM skills_job_dim) -- Important new line, ensures only jobs in the skills table are included
ORDER BY
    salary_year_avg DESC
LIMIT 10

)

SELECT 
    top_paying_jobs.*,
    skills
FROM top_paying_jobs
INNER JOIN skills_job_dim ON top_paying_jobs.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
ORDER BY
    salary_year_avg DESC
```

Because the filter happens before the `ORDER BY` and `LIMIT`, the two jobs with no skills are skipped and replaced by the next highest-paying ones. It was a good reminder to check how joins change the number of rows.

#### Analysing the fixed results
After fixing the query I exported the 10-job results, uploaded them to Claude again and asked for the same analysis:

<img width="793" height="679" alt="image" src="https://github.com/user-attachments/assets/b9d50349-4954-4276-8398-281a3556aff9" />

SQL now appears in 9 of 10 postings and Python in 8. Tableau is still the leading BI tool (6 of 10 postings vs 2 for Power BI), so the picture is much the same as before, though it's still a small sample. The 9th and 10th highest salaries are both $170,000, so which of the tied jobs makes the top 10 can vary between runs.

### 3. In-Demand Skills for Data Analysts
This query counts how often each skill appears across all UK data analyst postings, to show where the demand is.

```sql
SELECT
    skills,
    COUNT(skills_job_dim.job_id) AS demand_count
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst' AND
    job_country = 'United Kingdom'
GROUP BY
    skills
ORDER BY
    demand_count DESC
LIMIT 5;
```

| Skills | Demand Count |
|--------|-------------:|
| SQL | 4,480 |
| Excel | 4,281 |
| Power BI | 2,865 |
| Python | 2,129 |
| Tableau | 1,644 |

*Top 5 most requested skills in UK data analyst job postings*

**What I noticed:**
- SQL is the most requested skill, with Excel only just behind it (4,480 vs 4,281). Both are far ahead of everything else, so SQL and Excel look like the baseline for UK data analyst roles.
- Power BI comes third (2,865), well ahead of Tableau (1,644). That's the opposite of what I saw in the top-paying remote jobs in section 2, where Tableau led. Those were a small sample of remote roles, so it may just be that Power BI is more common across UK postings in general.
- Python is fourth (2,129), so it's useful but requested noticeably less often than SQL and Excel.

### 4. Skills Based on Salary
This one looks at the average salary for each skill across all data analyst postings with a listed salary, regardless of location, to see which skills are linked to higher pay.

```sql
SELECT
    skills,
    ROUND(AVG(salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
GROUP BY
    skills
ORDER BY
    avg_salary DESC
LIMIT 25;
```

| Skills | Average Salary ($) |
|--------|-------------------:|
| SVN | 400,000 |
| Solidity | 179,000 |
| Couchbase | 160,515 |
| DataRobot | 155,486 |
| Golang | 155,000 |
| MXNet | 149,000 |
| dplyr | 147,633 |
| VMware | 147,500 |
| Terraform | 146,734 |
| Twilio | 138,500 |

*Top 10 highest-paying skills for data analysts (the query returns the top 25)*

**What I noticed:**
- SVN is way out in front at $400,000, more than double the next skill. I didn't filter by number of postings, so averages like this could easily rest on one or two jobs. I'd treat the very top of the list with caution.
- Most of the top 25 are specialist tools rather than core analyst skills. There's a lot of software engineering and DevOps (Terraform, GitLab, Kafka, Puppet, Ansible, Airflow), machine learning (DataRobot, MXNet, Keras, PyTorch, Hugging Face, TensorFlow) and less common languages (Golang, Perl, Scala).
- SQL, Excel and Python, the skills in highest demand in section 3, don't appear in the top 25 at all. The most requested skills aren't the ones with the highest average salary.

### 5. Most Optimal Skills to Learn
For the last one I combined demand and salary to find skills that come up a lot and also pay well. It only looks at remote data analyst jobs with a listed salary, and I only included skills with more than 10 postings so a single job couldn't skew the average.

```sql
SELECT
    skills_dim.skill_id,
    skills_dim.skills,
    COUNT(skills_job_dim.job_id) AS demand_count,
    ROUND(AVG(job_postings_fact.salary_year_avg), 0) AS avg_salary
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND salary_year_avg IS NOT NULL
    AND job_work_from_home = True
GROUP BY
    skills_dim.skill_id
HAVING
    COUNT(skills_job_dim.job_id) > 10
ORDER BY
    avg_salary DESC,
    demand_count DESC
LIMIT 25;
```

| Skill ID | Skills | Demand Count | Average Salary ($) |
|----------|--------|-------------:|-------------------:|
| 8 | Go | 27 | 115,320 |
| 234 | Confluence | 11 | 114,210 |
| 97 | Hadoop | 22 | 113,193 |
| 80 | Snowflake | 37 | 112,948 |
| 74 | Azure | 34 | 111,225 |
| 77 | BigQuery | 13 | 109,654 |
| 76 | AWS | 32 | 108,317 |
| 4 | Java | 17 | 106,906 |
| 194 | SSIS | 12 | 106,683 |
| 233 | Jira | 20 | 104,918 |

*Top 10 skills by average salary (the query returns the top 25)*

**What I noticed:**
- Go has the highest average salary ($115,320) and decent demand (27 postings). Hadoop, Snowflake, Azure, AWS and BigQuery are also near the top, so cloud and data engineering tools seem to pay well.
- Looking at the full 25 results, the skills with by far the most demand are Python (236 postings), Tableau (230) and R (148), and they pay around $99,000 to $101,000. That's slightly lower than the cloud tools, but with a lot more jobs asking for them.
- The salaries across the whole list are quite close together, from about $97,600 to $115,300. Once I only counted skills with more than 10 postings, the huge gaps from section 4 mostly disappeared.
- SQL doesn't appear in the top 25 here, so its average salary is lower than these skills even though it's the most requested. SAS appears twice because it has two skill IDs (186 and 7) in the skills table.

# What I Learned
Some things I picked up while doing this project and the exercises around it:

- **CTEs and joins:** CTEs let me split a long query into smaller steps, and joins let me bring tables together to answer questions that no single table could.
- **Aggregation:** `GROUP BY`, `COUNT()` and `AVG()` turned thousands of rows into something I could actually read.
- **Check how joins change your row count:** I asked for 10 jobs and only got 8, because the `INNER JOIN` dropped two jobs with no skills listed. Filtering for skills inside the CTE fixed it, and I now check the row count after every join.
- **Combine first, filter once:** to get Q1 postings above a salary threshold, my first thought was to write the same `WHERE` clause for January, February and March separately. Instead I used `UNION ALL` to combine the three tables in a subquery and applied the `WHERE` and `ORDER BY` once. Small thing, but it changed how I think about structuring queries.
- **Joins aren't the only option:** I'd assumed you always needed a `JOIN` to filter one table using another. If you only need to check that a match exists, and don't need any columns from the other table, a subquery with `IN` does the job. Joins are for when you need columns from both.
- **Date functions:** extracting the month from a date and grouping by it showed that January to March had noticeably more postings than many of the later months. It was the first exercise that felt like proper analysis rather than just learning syntax.

# Conclusions

### Insights
1. **Top-paying jobs:** the top 10 remote data analyst roles pay between $184,000 and $650,000, though the highest looks like an outlier and most of the top salaries go with senior titles. In the UK the top 10 pay between $132,500 and $180,000, and most of them are not plain data analyst roles (Data Architect appears four times)
2. **Skills for top-paying jobs:** SQL appeared in 9 of the top 10 postings and Python in 8, with Tableau the leading BI tool (6 vs 2 for Power BI)
3. **Most in-demand skills:** SQL (4,480) and Excel (4,281) are the most requested skills in UK data analyst postings, followed by Power BI, Python and Tableau
4. **Skills with higher salaries:** the highest average salaries go with specialist skills such as SVN, Solidity and Couchbase, plus engineering and machine learning tools. These averages can rest on very few postings, and the most in-demand skills don't appear in the top 25
5. **Optimal skills:** Python and Tableau combine very high demand with salaries around $100,000, while cloud and data engineering tools such as Snowflake, Azure and AWS pay a bit more with lower demand

### Closing Thoughts
This project improved my SQL and gave me a better idea of what to learn next. Power BI and Tableau were both near the top for demand, so I want to start learning them, and I also want to get more confident with common table expressions and subqueries.

I'm posting about my SQL learning on [LinkedIn](https://www.linkedin.com/in/cal-hunter-1aa982239/), and feedback is welcome.
