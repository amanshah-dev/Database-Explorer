create function trg_after_course_invitation_create() returns trigger
    language plpgsql
as
$$
DECLARE
    v_user_id INT;
BEGIN
    /* Check if the inserted row has the specified "centreId" and "courseId" */
    IF NEW."centreId" IS NOT NULL AND NEW."courseId" = ANY(ARRAY[3323, 4215]) THEN
        /* Find the user with the matching email, ignoring case and whitespace */
        SELECT "id" INTO v_user_id
        FROM "User"
        WHERE "User"."email" = TRIM(LOWER(NEW."email"))
        LIMIT 1;

        /* If a matching user is found, create an entry in "UserCentre" */
        IF v_user_id IS NOT NULL THEN
            INSERT INTO "UserCentre" ("userId", "centreId", "createdAt", "updatedAt")
            VALUES (v_user_id, NEW."centreId", NOW(), NOW()) ON CONFLICT ("userId") do nothing;
        END IF;
    END IF;

    RETURN NEW;
END;
$$;

alter function trg_after_course_invitation_create() owner to neetprep_rw;
