create function process_complimentary_courses_for_abhyas2_recent_users() returns void
    language plpgsql
as
$$
DECLARE
    user_record RECORD;
    selected_course_offer_id INT;
BEGIN
    -- Identify users who bought course with ID 4775 in the last 2 days with valid expiry
    FOR user_record IN
        SELECT "userId", "expiryAt"
        FROM "UserCourse"
        WHERE "courseId" = 4775
        AND "startedAt" >= NOW() - INTERVAL '2 days'
        AND "expiryAt" - "startedAt" > INTERVAL '10 days'
    LOOP
        -- Determine the appropriate courseOfferId based on the expiry year
        IF EXTRACT(YEAR FROM user_record."expiryAt") = 2026 THEN
            selected_course_offer_id := 11829640; -- on prod
            -- selected_course_offer_id := 990834; -- on dev
        ELSIF EXTRACT(YEAR FROM user_record."expiryAt") = 2025 THEN
            selected_course_offer_id := 11826193; -- on prod
            -- selected_course_offer_id := 990833; -- on dev
        ELSE
            CONTINUE; -- Skip if expiry year is not 2026 or 2025
        END IF;

        -- Call the complimentary courses processing function for each user with determined courseOfferId
        PERFORM process_complimentary_courses(4775, selected_course_offer_id, user_record."userId");
    END LOOP;
END;
$$;

alter function process_complimentary_courses_for_abhyas2_recent_users() owner to neetprep_rw;
