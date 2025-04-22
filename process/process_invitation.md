create function process_invitation(invitation_id integer) returns void
    language plpgsql
as
$$
DECLARE
    invitation RECORD;
    user_course RECORD;
    users RECORD;
    final_user RECORD;
    course RECORD;
    days_to_expiry INT;
    currentUserCourseId INT;
BEGIN
    final_user = NULL;
    -- Fetch the invitation details
    SELECT * INTO invitation FROM "CourseInvitation" WHERE "id" = invitation_id;

    IF invitation."accepted" THEN
        -- Check if a UserCourse exists for this invitation
        SELECT * INTO user_course
        FROM "UserCourse"
        WHERE "invitationId" = invitation."id";

        IF FOUND THEN
            -- Update expiryAt and role if needed
            IF user_course."expiryAt" IS NULL THEN
                UPDATE "UserCourse"
                SET "expiryAt" = invitation."expiryAt"
                WHERE "id" = user_course."id";
            END IF;

            IF user_course."role"::text != invitation."role"::text THEN
                UPDATE "UserCourse"
                SET "role" = (invitation."role"::text)::"enum_UserCourse_role"
                WHERE "id" = user_course."id";
            END IF;
        END IF;
        RETURN;
    END IF;

    -- Calculate TTL for the invitation in days
    SELECT EXTRACT(EPOCH FROM (invitation."expiryAt" - invitation."createdAt")) / (60 * 60 * 24)
    INTO days_to_expiry;

    -- Fetch potential users
    IF days_to_expiry < 5 THEN
        -- Consider only email for users
        FOR users IN
            SELECT u."id" as "userId", u."email" as "email", u."phone" as "phone", uc."id" as "userCourseId"
            FROM "User" u
            LEFT JOIN "UserCourse" uc ON uc."userId" = u."id"
                AND uc."startedAt" <= CURRENT_TIMESTAMP
                AND uc."expiryAt" >= CURRENT_TIMESTAMP
            WHERE u."email" = invitation."email"
        LOOP
            final_user := users;
            EXIT;
        END LOOP;
    ELSE
        -- Consider email and phone for users
        FOR users IN
            SELECT u."id" as "userId", u."email" as "email", u."phone" as "phone", uc."id" as "userCourseId"
            FROM "User" u
            LEFT JOIN "UserCourse" uc ON uc."userId" = u."id"
                AND uc."startedAt" <= CURRENT_TIMESTAMP
                AND uc."expiryAt" >= CURRENT_TIMESTAMP
            WHERE u."email" = invitation."email"
               OR u."phone" = (
                  CASE
                    WHEN POSITION('+' IN invitation."phone") > 0 THEN invitation."phone"
                    ELSE '+91' || RIGHT(invitation."phone", 10)
                  END
                )
        LOOP
            final_user := users;
            currentUserCourseId := users."userCourseId";
            IF currentUserCourseId IS NOT NULL AND final_user IS NOT NULL THEN
                final_user := users;
                EXIT;
            END IF;
        END LOOP;
    END IF;

    -- Fetch the course details
    SELECT * INTO course
    FROM "Course"
    WHERE "id" = invitation."courseId";

    IF final_user IS NULL THEN
      -- DO NOTHING BUT WE DO IT THIS WAY BECAUSE OF SPECIAL WAY NOT NULL CHECK IS DONE. It returns TRUE if, and only if, every single column is NOT NULL.
    ELSE
        IF (final_user."email" = invitation."email" OR final_user."phone" = invitation."phone") THEN
            -- Create a new UserCourse
            INSERT INTO "UserCourse" (
                "userId", "courseId", "expiryAt", "startedAt", "role", "invitationId", "createdAt", "updatedAt"
            ) VALUES (
                final_user."userId", invitation."courseId", invitation."expiryAt", CURRENT_TIMESTAMP, (invitation."role"::text)::"enum_UserCourse_role",
                invitation."id", CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
            ) ON CONFLICT ("userId", "courseId", "expiryAt") DO NOTHING;

            -- Mark the invitation as accepted
            UPDATE "CourseInvitation"
            SET "accepted" = TRUE
            WHERE "id" = invitation."id";
        END IF;
    END IF;

END;
$$;

alter function process_invitation(integer) owner to neetprep_rw;
