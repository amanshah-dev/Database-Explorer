CREATE FUNCTION is_currently_paid_user(input_params JSONB) RETURNS BOOLEAN
LANGUAGE plpgsql
AS
$$
DECLARE
    paid_course_count INT;
    userId INT;
BEGIN
    -- Extract userId from input_params
    userId := input_params->>'user_id';

    -- Check if the user has any courses in the UserCourse table with duration greater than 10 days
    SELECT COUNT(*)
    INTO paid_course_count
    FROM "UserCourse" uc
    WHERE uc."userId" = userId 
      AND uc."startedAt" <= NOW() 
      AND uc."expiryAt" >= NOW()
      AND (uc."expiryAt" - uc."startedAt") > INTERVAL '10 days';

    -- Return true if there is at least one course meeting the conditions, otherwise false
    RETURN paid_course_count > 0;
END;
$$;

ALTER FUNCTION is_currently_paid_user(JSONB) OWNER TO learner;
