-- Create function to check if a user has any active target-based paid courses
create function check_is_active_target_paid_user(userid integer) returns boolean
    language plpgsql
as
$$
DECLARE
    active_course_count INT;
    target_based_course_ids INT[];  -- Array to store target-based course IDs
BEGIN
    -- Retrieve the target based course IDs from the Constant table
    SELECT ARRAY(
        SELECT (jsonb_array_elements_text(c."value"::jsonb))::INT
        FROM "Constant" c
        WHERE c."key" = 'TARGET_BASED_COURSE_IDS'
    )
    INTO target_based_course_ids;

    -- Check if the user has any active courses with a duration greater than 10 days and courseId in target based course IDs
    SELECT COUNT(*)
    INTO active_course_count
    FROM "UserCourse" uc
    WHERE uc."userId" = userId
    AND uc."expiryAt" > NOW()  -- The course must not have expired
    AND (uc."expiryAt" - uc."startedAt") > INTERVAL '2 days'  -- The course must be active for more than 2 days
    AND uc."courseId" = ANY(target_based_course_ids);  -- The course must be a target-based course

    -- Return true if there is at least one active course meeting the conditions, otherwise false
    RETURN active_course_count > 0;
END;
$$;

-- Alter function owner
alter function check_is_active_target_paid_user(integer) owner to neetprep_rw;
