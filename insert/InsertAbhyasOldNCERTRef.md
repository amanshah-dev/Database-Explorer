CREATE FUNCTION "InsertAbhyasOldNCERTRef"(newtestid INTEGER, oldtestid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
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
      WHERE "Test"."id" = oldTestId
  ), "TestQue" AS (
      SELECT "questionId", row_number() OVER(ORDER BY "seqNum" ASC, "questionId" ASC) AS "queNum", "seqNum"
      FROM "TestQuestion"
      WHERE "testId" = oldTestId
      ORDER BY "seqNum" ASC, "questionId" ASC
  )
  INSERT INTO "TestQuestionDetail" ("testQuestionId", "note")
  SELECT "NewTestQuestion"."id", "section" || ' - (OLD NCERT)'
  FROM "TestQue", "TestSection", "TestQuestion" AS "NewTestQuestion"
  WHERE "TestQue"."queNum" >= "TestSection"."startNum"
    AND "TestQue"."queNum" <= "TestSection"."endNum"
    AND "NewTestQuestion"."testId" = newTestId
    AND "NewTestQuestion"."questionId" = "TestQue"."questionId"
  ON CONFLICT ("testQuestionId") DO NOTHING;
END
$$;

ALTER FUNCTION "InsertAbhyasOldNCERTRef"(INTEGER, INTEGER) OWNER TO learner;
