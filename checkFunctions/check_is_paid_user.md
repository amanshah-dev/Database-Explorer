-- Create function to check if a user has any paid courses with a duration greater than 10 days
create function check_is_paid_user(input_params jsonb) returns boolean
    language plpgsql
as
$$
DECLARE
    paid_course_count INT;
    userId INT;  -- Declare a variable to hold the userId from the input parameters
BEGIN
    -- Extract userId from input_params
    userId := (input_params->>'user_id')::INT;

    -- Check if the user has any courses in the UserCourse table with duration greater than 10 days
    SELECT COUNT(*)
    INTO paid_course_count
    FROM "UserCourse" uc
    WHERE uc."userId" = userId
    AND (uc."expiryAt" - uc."startedAt") > INTERVAL '10 days';  -- Check for courses with more than 10 days duration

    -- Return true if there is at least one course meeting the conditions, otherwise false
    RETURN paid_course_count > 0;
END;
$$;

-- Alter function owner
alter function check_is_paid_user(jsonb) owner to neetprep_rw;
