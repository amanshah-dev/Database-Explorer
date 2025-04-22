CREATE FUNCTION give_access_to_complimentary_addons(p_course_id integer) 
RETURNS void
    LANGUAGE plpgsql
AS
$$
DECLARE
    target_courses INT[];
    c INT;
BEGIN
    -- Collect addon course IDs into the array
    SELECT array_agg("addonCourseId") INTO target_courses
    FROM "ComplimentaryAddon"
    WHERE "courseId" = p_course_id;

    -- If no addons found, exit early
    IF target_courses IS NULL THEN
        RETURN;
    END IF;

    -- For each user with course p_course_id, add the target courses
    FOR c IN SELECT unnest(target_courses) LOOP
            INSERT INTO "UserCourse" (
                "startedAt",
                "expiryAt",
                "role",
                "courseId",
                "userId",
                "couponId",
                "trial",
                "invitationId",
                "courseOfferId"
            )
            SELECT
                uc."startedAt",
                uc."expiryAt",
                uc."role",
                c as "courseId",
                uc."userId",
                uc."couponId",
                uc."trial",
                uc."invitationId",
                uc."courseOfferId"
            FROM "UserCourse" uc
            WHERE uc."courseId" = p_course_id
              AND uc."startedAt" < NOW()
              AND uc."expiryAt" > NOW()
              -- Skip if this user already has access to this course
              AND NOT EXISTS (
                SELECT 1
                FROM "UserCourse" uc2
                WHERE uc2."userId" = uc."userId"
                  AND uc2."courseId" = c
                  AND uc2."expiryAt" >= uc."expiryAt"
            );
        END LOOP;
END $$;

ALTER FUNCTION give_access_to_complimentary_addons(integer) OWNER TO learner;
