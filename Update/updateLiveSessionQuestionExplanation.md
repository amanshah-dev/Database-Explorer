create function "updateLiveSessionQuestionExplanation"(testid integer) returns void
    language plpgsql
as
$$
BEGIN
    UPDATE "Question" 
    SET "explanation" = concat(
        "explanation", 
        '<p>Question No: <b>', c."rowNum", 
        '</b> from Live Session Link Below</p>', 
        "Test"."resultMsgHtml"
    )
    FROM "TestQuestion", "Test", 
        (SELECT "questionId", 
                row_number() OVER (ORDER BY "TestQuestion"."questionId" ASC) AS "rowNum" 
         FROM "TestQuestion" 
         WHERE "testId" = testId) AS c
    WHERE "testId" = testId
      AND "Test"."id" = "TestQuestion"."testId"
      AND "Question"."id" = "TestQuestion"."questionId"
      AND c."questionId" = "Question"."id"
      AND ("Question"."explanation" IS NULL OR "Question"."explanation" = '')
      AND "resultMsgHtml" LIKE '%youtu%';
END
$$;

alter function "updateLiveSessionQuestionExplanation"(integer) owner to learner;
