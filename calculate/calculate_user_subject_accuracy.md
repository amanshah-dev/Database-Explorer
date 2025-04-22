-- Create function to calculate the user's accuracy in multiple subjects
create function calculate_user_subject_accuracy(p_userid integer, p_subjectids integer[] DEFAULT ARRAY[53, 54, 55, 56], p_limit integer DEFAULT 1000) returns void
    language plpgsql
as
$$
DECLARE
    subjectId INT; -- Declare a variable to hold each subjectId during the loop
BEGIN
    -- Loop through each subject in the provided array
    FOR subjectId IN SELECT UNNEST(p_subjectIds) LOOP
        WITH LevelBoundaries AS (
            /*
            Generate levels in increments of 10 from 0 to 100.
            */
            SELECT 0 AS "minLevel", 10 AS "maxLevel"
            UNION ALL SELECT 10, 20
            UNION ALL SELECT 20, 30
            UNION ALL SELECT 30, 40
            UNION ALL SELECT 40, 50
            UNION ALL SELECT 50, 60
            UNION ALL SELECT 60, 70
            UNION ALL SELECT 70, 80
            UNION ALL SELECT 80, 90
            UNION ALL SELECT 90, 100
        ),
        UserSubjectAnswers AS (
            /*
            Join Answer table with QuestionAnalytics and filter answers
            where the correctPercentage falls within a specific level range
            for the current subjectId.
            */
            SELECT
                "A"."userId",
                "Q"."subjectId",
                L."minLevel" AS "minLevel",
                L."maxLevel" AS "maxLevel",
                COUNT(DISTINCT("Q"."id")) AS "answerCount",
                COUNT(DISTINCT(CASE WHEN "A"."userAnswer" = "Q"."correctOptionIndex" THEN "Q"."id" ELSE NULL END)) AS "correctAnswerCount"
            FROM
                (SELECT * FROM "Answer" WHERE "Answer"."userId" = p_userId ORDER BY "createdAt" LIMIT p_limit) "A"
            INNER JOIN "QuestionAnalytics" AS "QA"
                ON "A"."questionId" = "QA"."id"
            INNER JOIN "Question" AS "Q"
                ON "A"."questionId" = "Q"."id"
            CROSS JOIN LevelBoundaries AS L
            WHERE "QA"."correctPercentage" >= L."minLevel"
              AND "QA"."correctPercentage" < L."maxLevel"
              AND "A"."userId" = p_userId
              AND "Q"."subjectId" = subjectId
            GROUP BY L."minLevel", L."maxLevel", "A"."userId", "Q"."subjectId"
        ),
        UserSubjectAccuracy AS (
            /*
            Calculate accuracy percentage for the given userId and current subject.
            */
            SELECT
                "userId",
                "subjectId",
                "minLevel",
                "maxLevel",
                CASE
                    WHEN "answerCount" = 0 THEN 0
                    ELSE ROUND((("correctAnswerCount"::FLOAT / "answerCount") * 100)::NUMERIC, 2)
                END AS "accuracyPercentage",
                "correctAnswerCount",
                "answerCount"
            FROM UserSubjectAnswers
        )
        -- Insert or Update results into the UserSubjectAccuracyByLevel table
        INSERT INTO "UserSubjectAccuracyByLevel" ("userId", "subjectId", "minLevel", "maxLevel", "accuracyPercentage", "correctAnswerCount", "answerCount")
        SELECT
            "userId",
            "subjectId",
            "minLevel",
            "maxLevel",
            "accuracyPercentage",
            "correctAnswerCount",
            "answerCount"
        FROM UserSubjectAccuracy
        ON CONFLICT ("userId", "subjectId", "minLevel", "maxLevel")
        DO UPDATE SET
            "accuracyPercentage" = EXCLUDED."accuracyPercentage",
            "correctAnswerCount" = EXCLUDED."correctAnswerCount",
            "answerCount" = EXCLUDED."answerCount",
            "updatedAt" = now();
    END LOOP;
END;
$$;

-- Alter function owner
alter function calculate_user_subject_accuracy(integer, integer[], integer) owner to learner;
