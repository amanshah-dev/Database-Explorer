CREATE FUNCTION grantfreetestseriesonpurchase(
    p_source_course_id INTEGER, 
    p_target_course_id INTEGER, 
    p_cutoff_date TIMESTAMP WITH TIME ZONE DEFAULT '2025-04-15 23:59:59+00'::TIMESTAMP WITH TIME ZONE
) 
RETURNS INTEGER 
    LANGUAGE plpgsql 
AS
$$
DECLARE
    affected_rows INTEGER := 0;
BEGIN
    -- Insert free target course for eligible source course users whose course has not expired
    INSERT INTO "UserCourse" (
        "userId",
        "courseId",
        "startedAt",
        "expiryAt",
        "role",
        "createdAt",
        "updatedAt",
        "trial"
    )
    SELECT
        uc."userId",
        p_target_course_id AS "courseId",
        CURRENT_TIMESTAMP AS "startedAt",
        uc."expiryAt",
        'courseStudent'::"enum_UserCourse_role" AS "role",
        CURRENT_TIMESTAMP AS "createdAt",
        CURRENT_TIMESTAMP AS "updatedAt",
        FALSE AS "trial"
    FROM "UserCourse" uc
    WHERE uc."courseId" = p_source_course_id
      AND uc."createdAt" < p_cutoff_date
      AND (uc."expiryAt" IS NULL OR uc."expiryAt" > CURRENT_TIMESTAMP)
      AND NOT EXISTS (
        SELECT 1
        FROM "UserCourse" uc2
        WHERE uc2."userId" = uc."userId"
          AND uc2."courseId" = p_target_course_id
          AND uc2."expiryAt" = uc."expiryAt"
    );

    -- Get the number of rows affected
    GET DIAGNOSTICS affected_rows = ROW_COUNT;

    RETURN affected_rows;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error in grantFreeTestSeriesOnPurchase: %', SQLERRM;
        RETURN -1;
END;
$$;

ALTER FUNCTION grantfreetestseriesonpurchase(integer, integer, timestamp with time zone) OWNER TO learner;
