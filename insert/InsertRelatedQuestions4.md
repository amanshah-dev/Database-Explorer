CREATE FUNCTION "InsertRelatedQuestions4"(lb INTEGER, ub INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN
        SELECT q.id 
        FROM "Question" q 
        LEFT JOIN "RelatedQuestion" rq ON q."id" = rq."questionId" 
        JOIN "QuestionSubTopic" qs ON q."id" = qs."questionId" 
        WHERE rq."questionId" IS NULL 
        AND q."id" BETWEEN lb AND ub
    LOOP
        PERFORM "InsertRelatedQuestions4"(rec.id, lb, ub);
    END LOOP;
END
$$;

ALTER FUNCTION "InsertRelatedQuestions4"(INTEGER, INTEGER) OWNER TO learner;
