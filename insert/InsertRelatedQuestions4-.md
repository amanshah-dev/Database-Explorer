CREATE FUNCTION "InsertRelatedQuestions4"(questionid INTEGER, lb INTEGER, ub INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
WITH "SubTopicQuestions" AS (
    SELECT
        q."id" AS "main_question_id",
        sub."subTopicId"
    FROM
        "Question" q
    JOIN
        "QuestionSubTopic" sub ON q."id" = sub."questionId"
    WHERE
        q."id" = questionId
),
-- Step 2: Find related questions for the subtopic of the specific questionId
"RelatedQuestions" AS (
    SELECT
        sq."main_question_id" AS "questionId",
        sub."questionId" AS "relatedQuestionId"
    FROM
        "SubTopicQuestions" sq
    JOIN
        "QuestionSubTopic" sub
        ON sq."subTopicId" = sub."subTopicId"
        AND sub."questionId" != sq."main_question_id"
        AND sub."questionId" NOT IN (
            SELECT q.id FROM "Question" q LEFT JOIN "RelatedQuestion" rq ON q."id" = rq."questionId" JOIN "QuestionSubTopic" qs ON q."id" = qs."questionId" WHERE rq."questionId" IS NULL AND q."id" BETWEEN lb AND ub
        )
        AND sub."questionId" NOT IN (SELECT "relatedQuestionId" FROM "RelatedQuestion" WHERE "questionId" = "main_question_id")
        AND (
            (sub."questionId" < sq."main_question_id" AND sub."questionId" NOT IN (SELECT "questionId1" FROM "DuplicateQuestion" WHERE "questionId2" = sq."main_question_id"))
            OR (sub."questionId" > sq."main_question_id" AND sub."questionId" NOT IN (SELECT "questionId2" FROM "DuplicateQuestion" WHERE "questionId1" = sq."main_question_id"))
        )
),
-- Step 3: Calculate similarity scores
"SimilarityScores" AS (
    SELECT
        rq."questionId",
        rq."relatedQuestionId",
        similarity(q1."question", q2."question") AS "score"
    FROM
        "RelatedQuestions" rq
    JOIN
        "Question" q1 ON rq."questionId" = q1."id" AND q1."deleted" = false
    JOIN
        "Question" q2 ON rq."relatedQuestionId" = q2."id" AND q2."deleted" = false
    WHERE similarity(q1."question", q2."question") > 0.3
    ORDER BY "score" DESC LIMIT (10 - (
        SELECT COUNT(*)
        FROM "RelatedQuestion"
        WHERE "questionId" = questionId AND "relatedQuestionId" NOT IN (
            SELECT q.id FROM "Question" q LEFT JOIN "RelatedQuestion" rq ON q."id" = rq."questionId" JOIN "QuestionSubTopic" qs ON q."id" = qs."questionId" WHERE rq."questionId" IS NULL AND q."id" BETWEEN lb AND ub
        )
    ))
),
-- Step 4: Assign seqId based on the highest similarity score priority
"RankedSimilarity" AS (
    SELECT
        "questionId",
        "relatedQuestionId"
    FROM
        "SimilarityScores"
)
-- Insert the results into the RelatedQuestion table for the specific questionId
INSERT INTO "RelatedQuestion" ("questionId", "relatedQuestionId")
SELECT DISTINCT ON ("questionId", "relatedQuestionId")
    "questionId",
    "relatedQuestionId"
FROM
    "RankedSimilarity" ON CONFLICT ("questionId", "relatedQuestionId") DO NOTHING;

UPDATE "RelatedQuestion"
SET "seqId" = t."seqId"
FROM (
    SELECT
        "RelatedQuestion"."id",
        ROW_NUMBER() OVER (ORDER BY similarity(q1."question", q2."question") DESC, "relatedQuestionId" ASC) AS "seqId"
    FROM
        "RelatedQuestion",
        "Question" q1,
        "Question" q2
    WHERE
        "RelatedQuestion"."questionId" = q1."id"
        AND "RelatedQuestion"."relatedQuestionId" = q2."id"
        AND "RelatedQuestion"."questionId" = questionId
) t
WHERE
    t."id" = "RelatedQuestion"."id";
END
$$;

ALTER FUNCTION "InsertRelatedQuestions4"(INTEGER, INTEGER, INTEGER) OWNER TO learner;
