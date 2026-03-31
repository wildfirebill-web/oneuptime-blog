# How to Use ClickHouse for Education Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Education Analytics, Learning Management, Student Performance, EdTech

Description: Build education analytics with ClickHouse to track student engagement, learning outcomes, course completion rates, and content effectiveness at scale.

---

## Education Analytics Use Cases

EdTech platforms and learning management systems generate event streams from video plays, quiz attempts, forum posts, and assignment submissions. ClickHouse handles this high-volume append-only data and delivers fast queries for student progress dashboards, cohort analysis, and content effectiveness reports.

## Learning Events Schema

```sql
CREATE TABLE learning_events
(
    event_time DateTime,
    user_id UInt64,
    course_id UInt32,
    content_id UInt64,
    content_type LowCardinality(String),
    event_type LowCardinality(String),
    duration_sec UInt32,
    score Float32,
    institution_id UInt32,
    grade_level UInt8,
    country LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (institution_id, course_id, user_id, event_time);
```

## Course Completion Funnel

```sql
SELECT
    course_id,
    countIf(event_type = 'enrollment') AS enrolled,
    countIf(event_type = 'first_lesson_complete') AS started,
    countIf(event_type = 'midpoint_reached') AS halfway,
    countIf(event_type = 'course_complete') AS completed,
    round(countIf(event_type = 'course_complete') /
          nullIf(countIf(event_type = 'enrollment'), 0) * 100, 1) AS completion_rate_pct
FROM learning_events
WHERE event_time >= today() - 180
GROUP BY course_id
ORDER BY enrolled DESC
LIMIT 20;
```

## Student Engagement Score

```sql
-- Weekly active engagement by student
SELECT
    user_id,
    toStartOfWeek(event_time) AS week,
    count() AS total_events,
    sum(duration_sec) / 3600.0 AS study_hours,
    countDistinct(course_id) AS active_courses,
    countIf(event_type = 'quiz_attempt') AS quiz_attempts,
    avg(score) AS avg_quiz_score
FROM learning_events
WHERE event_time >= today() - 84  -- 12 weeks
  AND event_type IN ('video_play', 'reading_complete', 'quiz_attempt', 'assignment_submit')
GROUP BY user_id, week
ORDER BY user_id, week;
```

## At-Risk Student Detection

```sql
-- Students with declining engagement in the last 2 weeks vs previous 2 weeks
WITH recent AS (
    SELECT user_id, course_id, count() AS events
    FROM learning_events
    WHERE event_time >= today() - 14
    GROUP BY user_id, course_id
),
prior AS (
    SELECT user_id, course_id, count() AS events
    FROM learning_events
    WHERE event_time BETWEEN today() - 28 AND today() - 14
    GROUP BY user_id, course_id
)
SELECT
    r.user_id,
    r.course_id,
    r.events AS recent_events,
    p.events AS prior_events,
    round((r.events - p.events) / nullIf(p.events, 0) * 100, 0) AS change_pct
FROM recent AS r
JOIN prior AS p ON r.user_id = p.user_id AND r.course_id = p.course_id
WHERE r.events < p.events * 0.5  -- engagement dropped by more than 50%
ORDER BY change_pct;
```

## Content Effectiveness

```sql
-- Which content pieces drive the highest quiz scores
SELECT
    content_id,
    content_type,
    count() AS students,
    avg(score) AS avg_score,
    avg(duration_sec) AS avg_time_spent_sec
FROM learning_events
WHERE event_type = 'quiz_attempt'
  AND score > 0
GROUP BY content_id, content_type
HAVING students >= 100
ORDER BY avg_score DESC
LIMIT 20;
```

## Summary

ClickHouse is well-suited for education analytics because learning event data is append-only, high-volume, and requires fast multi-dimensional aggregations across students, courses, and content. Build completion funnels, engagement scoring, at-risk detection, and content effectiveness reports using the MergeTree engine with partitioning by month and sorting by institution, course, and user.
