CREATE FUNCTION get_free_trials_remaining(user_id INTEGER, dpp_type TEXT)
    RETURNS JSON
    LANGUAGE plpgsql
AS
$$
DECLARE
    is_paid BOOLEAN;
    free_daily_practice_dpp INT;
    free_daily_revision_dpp INT;
    dpp_type_id INT;
    remaining_revision INT;
    remaining_practice INT;
    reset_after_revision BIGINT;
    reset_after_practice BIGINT;
    revision_count INT;
    practice_count INT;
    time_diff BIGINT;
BEGIN
    -- Check if the user is a paid user
    SELECT public.check_is_active_target_paid_user(user_id) INTO is_paid;
    
    IF is_paid THEN
        RETURN json_build_object(
            'revision', json_build_object(
                'total', -1,
                'remaining', -1,
                'resetAfter', 0
            ),
            'practice', json_build_object(
                'total', -1,
                'remaining', -1,
                'resetAfter', 0
            )
        );
    ELSE
        -- Fetch values from Constant table
        SELECT value::INT INTO free_daily_practice_dpp FROM "Constant" WHERE key = 'FREE_DAILY_PRACTICE_DPP';
        SELECT value::INT INTO free_daily_revision_dpp FROM "Constant" WHERE key = 'FREE_DAILY_REVISION_DPP';

        -- Set DPP type id based on dpp_type
        IF dpp_type = 'revision' THEN
            dpp_type_id := free_daily_revision_dpp;
        ELSIF dpp_type = 'practice' THEN
            dpp_type_id := free_daily_practice_dpp;
        END IF;

        -- Count revision DPPs created in the last 24 hours
        SELECT COUNT(*) INTO revision_count
        FROM "UserDpp"
        WHERE "userId" = user_id AND "dppType" = 'revision' AND "createdAt" > (CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::DATE;

        -- Count practice DPPs created in the last 24 hours
        SELECT COUNT(*) INTO practice_count
        FROM "UserDpp"
        WHERE "userId" = user_id AND "dppType" = 'practice' AND "createdAt" > (CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::DATE;

        -- Calculate remaining revisions
        remaining_revision := GREATEST(0, free_daily_revision_dpp - revision_count);

        -- Calculate remaining practices
        remaining_practice := GREATEST(0, free_daily_practice_dpp - practice_count);

        -- Calculate resetAfter timestamp for revision
        reset_after_revision := 0;
        IF remaining_revision = 0 THEN
            time_diff := EXTRACT(EPOCH FROM (CURRENT_DATE + INTERVAL '1 day')) * 1000 - EXTRACT(EPOCH FROM now()) * 1000;
            IF time_diff > 19800000 THEN
                reset_after_revision := time_diff - 19800000;
            ELSE
                reset_after_revision := 86400000 - 19800000 + time_diff;
            END IF;
        END IF;

        -- Calculate resetAfter timestamp for practice
        reset_after_practice := 0;
        IF remaining_practice = 0 THEN
            time_diff := EXTRACT(EPOCH FROM (CURRENT_DATE + INTERVAL '1 day')) * 1000 - EXTRACT(EPOCH FROM now()) * 1000;
            IF time_diff > 19800000 THEN
                reset_after_practice := time_diff - 19800000;
            ELSE
                reset_after_practice := 86400000 - 19800000 + time_diff;
            END IF;
        END IF;

        -- Build the JSON result
        RETURN json_build_object(
            'revision', json_build_object(
                'total', free_daily_revision_dpp,
                'remaining', remaining_revision,
                'resetAfter', reset_after_revision
            ),
            'practice', json_build_object(
                'total', free_daily_practice_dpp,
                'remaining', remaining_practice,
                'resetAfter', reset_after_practice
            )
        );
    END IF;
END;
$$;

ALTER FUNCTION get_free_trials_remaining(INTEGER, TEXT) OWNER TO neetprep_rw;
