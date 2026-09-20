# Task 2: What skills do the top-paying data analyst jobs ask for?

For this one I took the highest-paying data analyst jobs from my first query and looked at which skills they list. The idea is to see what's worth learning if you're aiming for the better-paid roles.

## The query

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

## Analysing the results

I exported the results as a CSV, uploaded it to Claude and asked it to go through the skills column and show me what stood out. Here's what it came back with:

<!-- screenshot goes here -->
![Claude's analysis](claude_analysis.png)

## Why I only got 8 jobs, not 10

I put `LIMIT 10` in the query but the CSV only had 8 jobs in it, which confused me at first.

The reason is the order things happen in. The CTE picks the top 10 by salary without caring whether a job has any skills listed. Then the `INNER JOIN` to `skills_job_dim` throws away any job with no skills rows. Two of my top 10 had no skills listed, so they got dropped and I was left with 8.

To fix it, I check for skills inside the CTE, so the `LIMIT` only counts jobs that have them:

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
        salary_year_avg IS NOT NULL AND
        EXISTS (
            SELECT 1 FROM skills_job_dim
            WHERE skills_job_dim.job_id = job_postings_fact.job_id
        )
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

Now it returns 10 jobs that all have skills, with the next highest-paying ones filling the gaps. I could have used a `LEFT JOIN` to keep the two jobs with no skills, but they'd just show `NULL` in the skills column, which doesn't help for this question.
