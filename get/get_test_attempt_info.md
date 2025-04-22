CREATE FUNCTION get_test_attempt_info(p_test_attempt_row "TestAttempt")
    RETURNS TABLE(test_name character varying, test_attempt_date timestamp with time zone, total_time_taken integer, total_questions integer)
    LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
        SELECT t."name"                                                                  AS "test_name",
               COALESCE(p_test_attempt_row."updatedAt", p_test_attempt_row."finishedAt") AS "test_attempt_date",
               (
                   CASE
                       WHEN p_test_attempt_row."userQuestionWiseDurationInSec" IS NULL OR
                            p_test_attempt_row."userQuestionWiseDurationInSec"::TEXT = '{}' THEN 0
                       ELSE (SELECT COALESCE(
                                            SUM(
                                                    CASE
                                                        WHEN elem.value ~ '^\d+$' THEN elem.value::INT
                                                        ELSE 0
                                                        END
                                            )::INT, 0
                                    )
                             FROM json_each_text(p_test_attempt_row."userQuestionWiseDurationInSec") AS elem)
                       END
                   )                                                                     AS "total_time_taken",
               t."numQuestions"                                                          AS "total_questions"
        FROM "Test" t
        WHERE t."id" = p_test_attempt_row."testId";
END;
$$;

ALTER FUNCTION get_test_attempt_info("TestAttempt") OWNER TO learner;
