CREATE FUNCTION grantfreetestseriesonpurchase() 
RETURNS integer 
    LANGUAGE plpgsql 
AS
$$
DECLARE
    affected_rows INTEGER := 0;
BEGIN
    -- Insert free test series (course 5501) for eligible course 2135 users
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
        5501 AS "courseId",
        CURRENT_TIMESTAMP AS "startedAt",
        uc."expiryAt",
        'courseStudent'::"enum_UserCourse_role" AS "role",
        CURRENT_TIMESTAMP AS "createdAt",
        CURRENT_TIMESTAMP AS "updatedAt",
        FALSE AS "trial"
    FROM "UserCourse" uc
    WHERE uc."courseId" = 2135
      AND uc."createdAt" < '2025-03-31 23:59:59+00'
      AND NOT EXISTS (
          SELECT 1
          FROM "UserCourse" uc2
          WHERE uc2."userId" = uc."userId"
            AND uc2."courseId" = 5501
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

ALTER FUNCTION grantfreetestseriesonpurchase() OWNER TO learner;
