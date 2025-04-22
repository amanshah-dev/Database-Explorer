-- Create function to get assumed test attempt with answers
create function assumed_test_attempt_with_answers(p_userid integer, p_testid integer DEFAULT NULL::integer, p_testattemptid integer DEFAULT NULL::integer)
    returns TABLE(id integer, "testId" integer, "userId" integer, "elapsedDurationInSec" integer, "currentQuestionOffset" integer, completed boolean, "userAnswers" json, "userQuestionWiseDurationInSec" json, result json, "createdAt" timestamp with time zone, "updatedAt" timestamp with time zone, "visitedQuestions" json, "markedQuestions" json, "nextTargetScore" integer, "nextTargetDate" timestamp without time zone, "finishedAt" timestamp with time zone, "offlineTestAttemptId" integer)
    language plpgsql
as
$$
DECLARE
    v_createdAt TIMESTAMPTZ := CURRENT_TIMESTAMP;
    v_updatedAt TIMESTAMPTZ := CURRENT_TIMESTAMP;
BEGIN
    -- Case: Both p_testId and p_testAttemptId are NULL
    IF p_testId IS NULL AND p_testAttemptId IS NULL THEN
        RETURN QUERY
            SELECT
                NULL AS id,
                NULL AS "testId",
                p_userId AS "userId",
                0 AS "elapsedDurationInSec",
                0 AS "currentQuestionOffset",
                FALSE AS completed,
                '{}'::JSON AS "userAnswers",
                '{}'::JSON AS "userQuestionWiseDurationInSec",
                '{}'::JSON AS result,
                v_createdAt AS "createdAt",
                v_updatedAt AS "updatedAt",
                '{}'::JSON AS "visitedQuestions",
                '{}'::JSON AS "markedQuestions",
                NULL AS "nextTargetScore",
                NULL AS "nextTargetDate",
                NULL AS "finishedAt",
                NULL AS "offlineTestAttemptId";
        RETURN; -- Exit function early
    END IF;

    -- Case: p_testAttemptId is provided, fetch from TestAttempt
    IF p_testAttemptId IS NOT NULL THEN
        RETURN QUERY
            SELECT
                ta.id,
                ta."testId",
                ta."userId",
                ta."elapsedDurationInSec",
                ta."currentQuestionOffset",
                ta.completed,
                ta."userAnswers",
                ta."userQuestionWiseDurationInSec",
                ta.result,
                ta."createdAt",
                ta."updatedAt",
                ta."visitedQuestions",
                ta."markedQuestions",
                ta."nextTargetScore",
                ta."nextTargetDate",
                ta."finishedAt",
                ta."offlineTestAttemptId"
            FROM
                "TestAttempt" ta
            WHERE
                ta."userId" = p_userId AND
                ta.id = p_testAttemptId;
        RETURN;
    END IF;

    -- Case: p_testId is provided but no test attempt exists
    RETURN QUERY
        WITH AnswerData AS (
            SELECT
                a."questionId",
                a."userAnswer",
                a."durationInSec",
                q."correctOptionIndex",
                t."positiveMarks",
                t."negativeMarks",
                CASE
                    WHEN a."userAnswer" = q."correctOptionIndex" THEN t."positiveMarks"
                    ELSE -1 * t."negativeMarks"
                    END AS marks
            FROM
                "Answer" a
                    JOIN "TestQuestion" tq ON a."questionId" = tq."questionId"
                    JOIN "Question" q ON tq."questionId" = q."id"
                    JOIN "Test" t ON tq."testId" = t."id"
            WHERE
                a."userId" = p_userId
              AND tq."testId" = p_testId
              AND a."testAttemptId" IS NULL
        )
        SELECT
            0 AS id, -- Default value for id
            p_testId,
            p_userId,
            COALESCE(SUM(ad."durationInSec"), 0)::INTEGER AS "elapsedDurationInSec",
            0 AS "currentQuestionOffset",
            FALSE AS completed,
            json_object_agg(ad."questionId", COALESCE(ad."userAnswer", '0')) AS "userAnswers",
            json_object_agg(ad."questionId", COALESCE(ad."durationInSec", 0)) AS "userQuestionWiseDurationInSec",
            json_build_object(
                    'correctAnswerCount', SUM(CASE WHEN ad."userAnswer" = ad."correctOptionIndex" THEN 1 ELSE 0 END),
                    'incorrectAnswerCount', SUM(CASE WHEN ad."userAnswer" IS NOT NULL AND ad."userAnswer" <> ad."correctOptionIndex" THEN 1 ELSE 0 END),
                    'totalMarks', SUM(ad.marks)
            ) AS result,
            v_createdAt,
            v_updatedAt,
            '{}'::JSON AS "visitedQuestions",
            '{}'::JSON AS "markedQuestions",
            NULL::INTEGER AS "nextTargetScore",
            NULL::TIMESTAMP AS "nextTargetDate",
            NULL::TIMESTAMPTZ AS "finishedAt",
            NULL::INTEGER AS "offlineTestAttemptId"
        FROM AnswerData ad;
END;
$$;

-- Alter function owner
alter function assumed_test_attempt_with_answers(integer, integer, integer) owner to learner;
