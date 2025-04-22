CREATE FUNCTION "DeleteTestQuestionsAndUpdateTest"(testid INTEGER, questionids TEXT) 
    RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE
  ids INT[];
BEGIN
  -- Convert questionIds (text) to an array of integers
  ids = string_to_array(questionIds,',');

  -- Check if any questions exist for the given testId and questionIds
  IF (SELECT count(*) = 0 FROM "TestQuestion" WHERE "testId" = testId AND "questionId" = ANY(ids)) THEN
    RETURN;
  END IF;

  -- Define CTE to retrieve sections and the range of question indices
  WITH "TestSection" AS (
    SELECT
      "Test"."id" AS "testId",
      ordinality AS element_index,
      value AS json_element,
      value->>0 AS "section",
      (value->>1)::INTEGER AS "startNum",
      (COALESCE((LEAD((value->>1)) OVER())::INTEGER, "numQuestions" + 1) - 1)::INTEGER AS "endNum",
      value->>2 AS "attemptLimit"
    FROM
      "Test",
      json_array_elements("Test"."sections") WITH ORDINALITY
    WHERE "Test"."id" = testId
  ),
  "TestQue" AS (
    SELECT "questionId", 
           row_number() OVER (ORDER BY "seqNum" ASC, "questionId" ASC) AS "queNum", 
           "seqNum"
    FROM "TestQuestion"
    WHERE "testId" = testId
    ORDER BY "seqNum" ASC, "questionId" ASC
  ),
  "NewTestSection" AS (
    SELECT "questionId", 
           "section", 
           row_number() OVER (ORDER BY "queNum" ASC) AS "newQueNum"
    FROM "TestSection", "TestQue"
    WHERE "queNum" >= "startNum" 
      AND "queNum" <= "endNum" 
      AND NOT("questionId" = ANY(ids))
  ),
  "NewSection" AS (
    SELECT '["' || "section" || '", ' || min("newQueNum") || ']' AS "newSections", 
           min("newQueNum")
    FROM "NewTestSection"
    GROUP BY "section"
    ORDER BY min("newQueNum")
  )
  -- Update Test table with the new sections and question count
  UPDATE "Test"
  SET 
    "sections" = (SELECT '[' || string_agg("newSections", ', ') || ']' 
                  FROM "NewSection")::json,
    "numQuestions" = (SELECT count(*) 
                      FROM "TestQuestion", "Question"
                      WHERE "testId" = "Test"."id" 
                        AND "Question"."id" = "TestQuestion"."questionId" 
                        AND "Question"."deleted" = FALSE 
                        AND NOT("questionId" = ANY(ids)))
  WHERE "id" = testId;

  -- Delete the selected questions from the TestQuestion table
  DELETE FROM "TestQuestion" 
  WHERE "testId" = testId 
    AND "questionId" = ANY(ids);

END
$$;

ALTER FUNCTION "DeleteTestQuestionsAndUpdateTest"(INTEGER, TEXT) OWNER TO learner;
