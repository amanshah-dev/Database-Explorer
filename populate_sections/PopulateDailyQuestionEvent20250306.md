create function "PopulateDailyQuestionEvent20250306"(datelower timestamp with time zone DEFAULT CURRENT_DATE) returns void
    language plpgsql
as
$$
DECLARE today DATE;
BEGIN 
    INSERT INTO "DailyQuestionEvent" (
      "questionId", "eventCount", "eventType", 
      "eventDate"
    )
    WITH cte_answer AS (
      SELECT 
        ques.id as "questionId", 
        COUNT(
          CASE WHEN (
            ques."correctOptionIndex" = ans."userAnswer"
          ) THEN 1 ELSE NULL :: INTEGER END
        ) AS "correctAnswerCount", 
        COUNT(
          CASE WHEN (
            ques."correctOptionIndex" <> ans."userAnswer"
          ) THEN 1 ELSE NULL :: INTEGER END
        ) AS "incorrectAnswerCount", 
        -- OPTION RELATED
        COUNT(
          CASE WHEN (ans."userAnswer" = 0) THEN 1 ELSE NULL :: INTEGER END
        ) AS "option1AnswerCount", 
        COUNT(
          CASE WHEN (ans."userAnswer" = 1) THEN 1 ELSE NULL :: INTEGER END
        ) AS "option2AnswerCount", 
        COUNT(
          CASE WHEN (ans."userAnswer" = 2) THEN 1 ELSE NULL :: INTEGER END
        ) AS "option3AnswerCount", 
        COUNT(
          CASE WHEN (ans."userAnswer" = 3) THEN 1 ELSE NULL :: INTEGER END
        ) AS "option4AnswerCount", 
        -- REASON RELATED
        COUNT(
          CASE WHEN (
            (ans."incorrectAnswerReason"):: text = '1' :: text
          ) THEN 1 ELSE NULL :: integer END
        ) AS "incorrectReason1Count", 
        COUNT(
          CASE WHEN (
            (ans."incorrectAnswerReason"):: text = '2' :: text
          ) THEN 1 ELSE NULL :: integer END
        ) AS "incorrectReason2Count", 
        COUNT(
          CASE WHEN (
            (ans."incorrectAnswerReason"):: text = '3' :: text
          ) THEN 1 ELSE NULL :: integer END
        ) AS "incorrectReason3Count" 
      FROM 
        "Answer" ans 
        INNER JOIN "Question" ques ON ans."questionId" = ques.id 
      WHERE 
        ans."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
        AND ans."createdAt" >= (datelower - interval '5.5 hours')
        AND ans."createdAt" < (datelower + interval '1 day' - interval '5.5 hours') 
        AND ques.type = ANY(
          ARRAY[ 'MCQ-SO' :: public."enum_Question_type", 
          'MCQ-AR' :: public."enum_Question_type" ]
        ) 
      GROUP BY 
        ques.id
    ), 
    cte_bookmark AS (
      SELECT 
        bq."questionId", 
        COUNT(id) AS "bookmarkCount" 
      FROM 
        "BookmarkQuestion" bq,
        -- redundant join but we need it as there is no foreign key constraint currently. so it can be removed when we have foreign key constraint added
        "Question" q
      WHERE
        q."id" = "BookmarkQuestion"."questionId"
        AND bq."createdAt" >= (datelower - interval '5.5 hours')
        AND bq."createdAt" < (datelower + interval '1 day' - interval '5.5 hours')
      GROUP BY 
        bq."questionId"
    ), 
    cte_issue AS (
      SELECT 
        cissue."questionId", 
        COUNT(cissue.id) AS "issueCount" 
      FROM 
        "CustomerIssue" cissue 
      WHERE 
        cissue."createdAt" >= (datelower - interval '5.5 hours')
        AND cissue."createdAt" < (datelower + interval '1 day' - interval '5.5 hours')
        AND cissue."questionId" IS NOT NULL 
      GROUP BY 
        cissue."questionId"
    ), 
    cte_doubts AS (
      SELECT 
        db."questionId", 
        COUNT(db.id) AS "doubtCount" 
      FROM 
        "Doubt" db 
      WHERE 
        db."createdAt" >= (datelower - interval '5.5 hours')
        AND db."createdAt" < (datelower + interval '1 day' - interval '5.5 hours')
        AND db."questionId" IS NOT NULL 
      GROUP BY 
        db."questionId"
    ) 
    select 
      "questionId", 
      "correctAnswerCount" AS "eventCount", 
      'correctAnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer
    where 
      "correctAnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "incorrectAnswerCount" AS "eventCount", 
      'incorrectAnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "incorrectAnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "option1AnswerCount" AS "eventCount", 
      'option1AnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "option1AnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "option2AnswerCount" AS "eventCount", 
      'option2AnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "option2AnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "option3AnswerCount" AS "eventCount", 
      'option3AnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "option3AnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "option4AnswerCount" AS "eventCount", 
      'option4AnswerCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "option4AnswerCount" > 0 
    union all 
    select 
      "questionId", 
      "incorrectReason1Count" AS "eventCount", 
      'incorrectReason1Count' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "incorrectReason1Count" > 0 
    union all 
    select 
      "questionId", 
      "incorrectReason2Count" AS "eventCount", 
      'incorrectReason2Count' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "incorrectReason2Count" > 0 
    union all 
    select 
      "questionId", 
      "incorrectReason3Count" AS "eventCount", 
      'incorrectReason3Count' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_answer 
    where 
      "incorrectReason3Count" > 0 
    union all 
    select 
      "questionId", 
      "bookmarkCount" AS "eventCount", 
      'bookmarkCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_bookmark 
    where 
      "bookmarkCount" > 0 
    union all 
    select 
      "questionId", 
      "issueCount" AS "eventCount", 
      'issueCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_issue 
    where 
      "issueCount" > 0 
    union all 
    select 
      "questionId", 
      "doubtCount" AS "eventCount", 
      'doubtCount' AS "eventType", 
      datelower::DATE AS "eventDate" 
    from 
      cte_doubts 
    where 
      "doubtCount" > 0 
    ON CONFLICT (
        "eventDate", "questionId", "eventType"
      ) DO 
    UPDATE 
    SET 
      "questionId" = EXCLUDED."questionId", 
      "eventDate" = EXCLUDED."eventDate", 
      "eventType" = EXCLUDED."eventType", 
      "eventCount" = EXCLUDED."eventCount";
END;
$$;

alter function "PopulateDailyQuestionEvent20250306"(timestamp with time zone) owner to learner;
