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

### 1. Top Paying Data Analyst Jobs
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

**What I noticed:**
- [Salary range of the top 10]
- [Which companies / industries appeared]
- [Anything surprising about the job titles]

![Top Paying Roles](assets/1_top_paying_roles.png)
*[Describe your chart and how you made it]*

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
This query counts how often each skill appears in remote data analyst postings.

```sql
SELECT
    skills,
    COUNT(skills_job_dim.job_id) AS demand_count
FROM job_postings_fact
INNER JOIN skills_job_dim ON job_postings_fact.job_id = skills_job_dim.job_id
INNER JOIN skills_dim ON skills_job_dim.skill_id = skills_dim.skill_id
WHERE
    job_title_short = 'Data Analyst'
    AND job_work_from_home = True
GROUP BY
    skills
ORDER BY
    demand_count DESC
LIMIT 5;
```

**What I noticed:** [Your takeaway in 2-3 sentences]

| Skills | Demand Count |
|--------|--------------|
| [ ]    | [ ]          |

### 4. Skills Based on Salary
This one looks at the average salary for each skill, to see which skills are linked to higher pay.

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
    AND job_work_from_home = True
GROUP BY
    skills
ORDER BY
    avg_salary DESC
LIMIT 25;
```

**What I noticed:** [Your takeaway: which kinds of skills sit at the top?]

| Skills | Average Salary ($) |
|--------|-------------------:|
| [ ]    | [ ]                |

### 5. Most Optimal Skills to Learn
For the last one I combined demand and salary to find skills that come up a lot and also pay well. I only included skills with more than 10 postings so a single job couldn't skew the average.

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

**What I noticed:** [Your takeaway: which skills hit the sweet spot?]

| Skill ID | Skills | Demand Count | Average Salary ($) |
|----------|--------|--------------|-------------------:|
| [ ]      | [ ]    | [ ]          | [ ]                |

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
1. **Top-paying jobs:** [Your finding]
2. **Skills for top-paying jobs:** SQL appeared in 9 of the top 10 postings and Python in 8, with Tableau the leading BI tool (6 vs 2 for Power BI)
3. **Most in-demand skills:** [Your finding]
4. **Skills with higher salaries:** [Your finding]
5. **Optimal skills:** [Your finding]

### Closing Thoughts
This project improved my SQL and gave me a better idea of what to learn next. [One or two sentences on what you'll do next.]

I'm posting about my SQL learning on [LinkedIn](https://www.linkedin.com/in/your-profile), and feedback is welcome.
