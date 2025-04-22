-- Create function to calculate results for temporary offline test attempts based on testId
create function "CalculateTempOfflineTestAttemptResults"(testid integer) returns void
    language plpgsql
as
$$
BEGIN
  WITH "TestQueWSeq" AS (
    -- Update answers
    SELECT "testId", "questionId", ROW_NUMBER() OVER(ORDER BY "TestQuestion"."seqNum" ASC, "TestQuestion"."questionId" ASC) AS "seqNo" 
    FROM "TestQuestion", "Question" 
    WHERE "testId" = testId 
      AND "TestQuestion"."questionId" = "Question"."id" 
      AND "Question"."deleted" = false
  )
  
  -- Update the answers for TempOfflineTestAttempt
  UPDATE "TempOfflineTestAttempt" 
  SET "quesAnswers" = t."jsonAnswers" 
  FROM (
    SELECT
      "TempOfflineTestAttempt"."id" AS "offlineTestAttemptId",
      jsonb_merge_agg(jsonb_build_object("questionId"::text, 
        CASE value::text 
          WHEN 'A' THEN '0' 
          WHEN 'B' THEN '1' 
          WHEN 'C' THEN '2' 
          WHEN 'D' THEN '3' 
        END)) AS "jsonAnswers"
    FROM
      "TempOfflineTestAttempt",
      jsonb_array_elements_text("answers") WITH ORDINALITY,
      "TestQueWSeq"
    WHERE
      "TestQueWSeq"."testId" = "TempOfflineTestAttempt"."rawTestId" 
      AND ordinality = "TestQueWSeq"."seqNo" 
      AND value::text IN ('A', 'B', 'C', 'D') 
    GROUP BY "TempOfflineTestAttempt"."id"
  ) t 
  WHERE t."offlineTestAttemptId" = "TempOfflineTestAttempt"."id";
  
  -- Update the results for TempOfflineTestAttempt
  UPDATE "TempOfflineTestAttempt" 
  SET "result" = t4."result" 
  FROM (
    SELECT "offlineTestAttemptId", 
           jsonb_build_object(
               'correctAnswerCount', SUM("correctAnswerCount"), 
               'incorrectAnswerCount', SUM("incorrectAnswerCount"), 
               'totalMarks', SUM("totalMarks"),
               'sections', json_agg(jsonb_build_object(
                   'sectionName', "sectionName", 
                   'totalMarks', "totalMarks", 
                   'correctAnswerCount', "correctAnswerCount", 
                   'incorrectAnswerCount', "incorrectAnswerCount"
               ))
           ) AS "result" 
    FROM (
      SELECT 
        "offlineTestAttemptId", 
        section AS "sectionName", 
        "sectionIndex", 
        SUM(t2."questionMarks") AS "totalMarks", 
        SUM(CASE WHEN "questionMarks" > 0 THEN 1 ELSE 0 END) AS "correctAnswerCount", 
        SUM(CASE WHEN "questionMarks" < 0 THEN 1 ELSE 0 END) AS "incorrectAnswerCount"
      FROM (
        SELECT 
          "TempOfflineTestAttempt"."id" AS "offlineTestAttemptId", 
          "sectionIndex", 
          t."testId", 
          t."questionId", 
          t."questionNum", 
          t.section, 
          COALESCE(t."attemptLimit"::INTEGER, t."endNum"::INTEGER - t."startNum"::INTEGER + 1) AS "attemptLimit", 
          CASE t1."queAnswer" 
            WHEN 'A' THEN 0 
            WHEN 'B' THEN 1 
            WHEN 'C' THEN 2 
            WHEN 'D' THEN 3 
          END AS "markedAnswer", 
          "Question"."correctOptionIndex" AS "correctAnswer", 
          COUNT(t1."queAnswer") OVER(PARTITION BY t.section, "TempOfflineTestAttempt"."id" ORDER BY t."questionNum") AS "attemptCount", 
          CASE 
            WHEN COUNT(t1."queAnswer") OVER(PARTITION BY t.section, "TempOfflineTestAttempt"."id" ORDER BY t."questionNum") <= COALESCE(t."attemptLimit"::INTEGER, t."endNum"::INTEGER - t."startNum"::INTEGER + 1) 
            THEN CASE 
                    WHEN CASE t1."queAnswer" 
                           WHEN 'A' THEN 0 
                           WHEN 'B' THEN 1 
                           WHEN 'C' THEN 2 
                           WHEN 'D' THEN 3 
                         END = "Question"."correctOptionIndex" 
                    THEN t."positiveMarks" 
                    WHEN t1."queAnswer" IS NOT NULL THEN t."negativeMarks" * -1 
                    ELSE 0 
                 END 
            ELSE 0 
          END AS "questionMarks" 
        FROM 
          "Test",
          json_array_elements("Test"."sections") WITH ORDINALITY,
          "TestQuestion"
        WHERE 
          "Test"."id" = testId 
          AND "TestQuestion"."testId" = "Test"."id"
      ) t, 
      "TempOfflineTestAttempt",
      jsonb_array_elements_text("answers") WITH ORDINALITY t1 ("queAnswer", "queIndex"), 
      "Question" 
      WHERE 
        "questionNum"::INTEGER >= "startNum"::INTEGER 
        AND "questionNum"::INTEGER <= "endNum"::INTEGER 
        AND "TempOfflineTestAttempt"."rawTestId" = t."testId" 
        AND t1."queIndex"::INTEGER = "questionNum"::INTEGER
        AND "Question"."id" = t."questionId" 
        AND "TempOfflineTestAttempt"."id" = offlinetestattemptid
      GROUP BY "offlineTestAttemptId", section, "sectionIndex" 
      ORDER BY "sectionIndex"
    ) t3 
    GROUP BY "offlineTestAttemptId"
  ) t4 
  WHERE t4."offlineTestAttemptId" = "TempOfflineTestAttempt"."id";
END
$$;

-- Alter function owner
alter function "CalculateTempOfflineTestAttemptResults"(integer) owner to neetprep_rw;
