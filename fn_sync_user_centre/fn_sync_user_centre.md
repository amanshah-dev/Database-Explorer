CREATE FUNCTION fn_sync_user_centre() RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    v_inserts INT;
BEGIN
    /* Insert into "UserCentre" for all matching entries at once and count inserts */
    WITH ins AS (
        INSERT INTO "UserCentre" ("userId", "centreId", "createdAt", "updatedAt")
        SELECT "User"."id", "CourseInvitation"."centreId", NOW(), NOW()
        FROM "CourseInvitation"
        JOIN "User" ON TRIM(LOWER("CourseInvitation"."email")) = "User"."email"
        WHERE "CourseInvitation"."centreId" IS NOT NULL AND "CourseInvitation"."courseId" = ANY(ARRAY[3323, 4215])
        ON CONFLICT("userId") DO NOTHING
        RETURNING 1
    )
    SELECT COUNT(*) INTO v_inserts FROM ins;

    RETURN v_inserts;
END;
$$;

ALTER FUNCTION fn_sync_user_centre() OWNER TO neetprep_rw;
