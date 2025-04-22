create function "SelectQuestionForRevisionTest"(userid integer, startpercentile integer, endpercentile integer, correct boolean, numquestions integer) 
    returns integer[] 
    language plpgsql 
as 
$$
DECLARE
    ids INTEGER[];
BEGIN
    IF (correct = TRUE) THEN
        WITH attempts AS (
            SELECT "questionId", ntile(100) OVER (ORDER BY "Answer"."createdAt" DESC) AS "percentile" 
            FROM "Answer", "Question" 
            WHERE "userId" = userId 
              AND "Question"."id" = "Answer"."questionId" 
              AND "Answer"."userAnswer" = "Question"."correctOptionIndex" 
              AND "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::timestamp with time zone 
              AND "Answer"."createdAt" >= NOW() - interval '6 months'
        )
        -- Force use "partial_idx_answer_created_at_1_month"
        SELECT array_agg("relatedQuestionId") INTO ids 
        FROM (
            SELECT "relatedQuestionId" 
            FROM attempts, "RelatedQuestion" 
            LEFT JOIN "QuestionAnalytics" qa1 ON qa1."id" = "RelatedQuestion"."questionId" 
            LEFT JOIN "QuestionAnalytics" qa2 ON "RelatedQuestion"."relatedQuestionId" = qa2."id" 
            WHERE "percentile" > startPercentile 
              AND "percentile" <= endPercentile 
              AND "RelatedQuestion"."questionId" = attempts."questionId" 
              AND "RelatedQuestion"."relatedQuestionId" != attempts."questionId" 
              AND ((qa2."correctPercentage" < qa1."correctPercentage") 
                   OR qa2."correctPercentage" IS NULL 
                   OR qa1."correctPercentage" IS NULL) 
            ORDER BY RANDOM() 
            LIMIT numQuestions
        ) t;
    ELSE
        WITH attempts AS (
            SELECT "questionId", ntile(100) OVER (ORDER BY "Answer"."createdAt" DESC) AS "percentile" 
            FROM "Answer", "Question" 
            WHERE "userId" = userId 
              AND "Question"."id" = "Answer"."questionId" 
              AND "Answer"."userAnswer" != "Question"."correctOptionIndex" 
              AND "Answer"."userAnswer" IS NOT NULL 
              AND "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::timestamp with time zone 
              AND "Answer"."createdAt" >= NOW() - interval '6 months'
            -- Force use "partial_idx_answer_created_at_1_month"
            AND NOT EXISTS (
                SELECT * 
                FROM "Answer" a 
                WHERE a."userId" = userId 
                  AND a."questionId" = "Answer"."questionId" 
                  AND a."userAnswer" = "Question"."correctOptionIndex"
            )
        )
        SELECT array_agg("relatedQuestionId") INTO ids 
        FROM (
            SELECT "relatedQuestionId" 
            FROM attempts, "RelatedQuestion" 
            LEFT JOIN "QuestionAnalytics" qa1 ON qa1."id" = "RelatedQuestion"."questionId" 
            LEFT JOIN "QuestionAnalytics" qa2 ON "RelatedQuestion"."relatedQuestionId" = qa2."id" 
            WHERE "percentile" > startPercentile 
              AND "percentile" <= endPercentile 
              AND "RelatedQuestion"."questionId" = attempts."questionId" 
              AND "RelatedQuestion"."relatedQuestionId" != attempts."questionId" 
              AND ((qa2."correctPercentage" > qa1."correctPercentage") 
                   OR qa2."correctPercentage" IS NULL 
                   OR qa1."correctPercentage" IS NULL) 
            ORDER BY RANDOM() 
            LIMIT numQuestions
        ) t;
    END IF;

    RETURN ids;
END;
$$;

alter function "SelectQuestionForRevisionTest"(integer, integer, integer, boolean, integer) owner to learner;
