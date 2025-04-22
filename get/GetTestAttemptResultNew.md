CREATE FUNCTION "GetTestAttemptResultNew"(testattemptid integer)
    RETURNS jsonb
    LANGUAGE plpgsql
AS
$$
DECLARE
  testid integer;
  userid integer;
  result JSONB;
BEGIN
  SELECT "userId", "testId" INTO userid, testid FROM "TestAttempt" WHERE "id" = testattemptid;
  
  WITH test_sections AS (
    SELECT "Test"."id" AS "testId",
      ordinality AS "sectionIndex",
      value AS json_element,
      value->>0 AS section,
      value->>1 AS "startNum",
      value->>2 AS "attemptLimit",
      "positiveMarks",
      "negativeMarks",
      CASE
        WHEN (COALESCE((LEAD((VALUE->>1)) OVER())::INTEGER, "numQuestions" + 1) - 1)::INTEGER = 0
        THEN "numQuestions"
        ELSE (COALESCE((LEAD((VALUE->>1)) OVER())::INTEGER, "numQuestions" + 1) - 1)::INTEGER
      END AS "endNum",
      DENSE_RANK() OVER (ORDER BY "TestQuestion"."seqNum" ASC, "TestQuestion"."questionId" ASC) AS "questionNum",
      "TestQuestion"."questionId" AS "questionId"
    FROM "Test"
      LEFT JOIN LATERAL json_array_elements(COALESCE("Test"."sections", '[]'::json)) WITH ORDINALITY ON TRUE
      JOIN "TestQuestion"
    ON "Test"."id" = testid AND
      "TestQuestion"."testId" = "Test"."id"
  ),
  test_attempt_sections AS (
    SELECT "TestAttempt"."id" AS "testAttemptId",
      "sectionIndex",
      test_sections."testId",
      test_sections."questionId",
      test_sections."questionNum",
      test_sections.section,
      COALESCE(test_sections."attemptLimit"::INTEGER, test_sections."endNum"::INTEGER - test_sections."startNum"::INTEGER + 1) AS "attemptLimit",
      t1."userAnswer" AS "markedAnswer",
      "Question"."correctOptionIndex" AS "correctAnswer",
      COUNT(t1."questionId") OVER(PARTITION BY test_sections.section, "TestAttempt"."id" ORDER BY test_sections."questionNum") AS "attemptCount",
      CASE
        WHEN COUNT(t1."questionId") OVER(PARTITION BY test_sections.section, "TestAttempt"."id" ORDER BY test_sections."questionNum") <= COALESCE(test_sections."attemptLimit"::INTEGER, test_sections."endNum"::INTEGER - test_sections."startNum"::INTEGER + 1)
        THEN
          CASE
            WHEN t1."userAnswer"::integer = "Question"."correctOptionIndex"
            THEN test_sections."positiveMarks"
            WHEN t1."userAnswer" IS NOT NULL
            THEN test_sections."negativeMarks" * -1
            ELSE 0
          END
        ELSE 0 END AS "questionMarks"
      FROM test_sections
        INNER JOIN "TestAttempt" ON "TestAttempt"."testId" = test_sections."testId" AND "TestAttempt"."userId" = userid AND "questionNum" >= "startNum"::integer AND "questionNum" <= "endNum"::integer
        INNER JOIN "Question" ON "Question"."id" = test_sections."questionId"
        LEFT OUTER JOIN json_each_text("userAnswers") t1 ("questionId", "userAnswer") ON t1."questionId"::INTEGER = test_sections."questionId"
  ),
  section_results AS (
    SELECT "testAttemptId",
      section AS "sectionName",
      "sectionIndex",
      SUM(test_attempt_sections."questionMarks") AS "totalMarks",
      SUM(CASE WHEN "questionMarks" > 0 THEN 1 ELSE 0 END) AS "correctAnswerCount",
      SUM(CASE WHEN "questionMarks" < 0 then 1 else 0 end) AS "incorrectAnswerCount"
    FROM test_attempt_sections
    GROUP BY "testAttemptId", section, "sectionIndex"
    ORDER BY "sectionIndex"
  ),
  test_results AS (
    SELECT "testAttemptId",
      JSONB_BUILD_OBJECT(
        'correctAnswerCount', SUM("correctAnswerCount"),
        'incorrectAnswerCount', SUM("incorrectAnswerCount"),
        'totalMarks', SUM("totalMarks"),
        'sections',
          JSON_AGG(
            JSONB_BUILD_OBJECT(
              'sectionName', "sectionName",
              'totalMarks', "totalMarks",
              'correctAnswerCount', "correctAnswerCount",
              'incorrectAnswerCount', "incorrectAnswerCount"
           )
        )
      ) AS "result"
    FROM section_results
    WHERE "testAttemptId" = testattemptid
    GROUP BY "testAttemptId"
  )
  SELECT test_results."result" INTO result FROM test_results;
  
  RETURN result;
END
$$;

ALTER FUNCTION "GetTestAttemptResultNew"(integer) OWNER TO neetprep_rw;
