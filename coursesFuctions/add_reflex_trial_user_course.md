-- Create function to add reflex trial for user and course
create function add_reflex_trial_user_course(user_id integer, course_id integer) returns void
    language plpgsql
as
$$
BEGIN
    -- Check if a UserCourse already exists for the given userId and courseId
    IF NOT EXISTS (
        SELECT 1
        FROM "UserCourse"
        WHERE "userId" = user_id AND "courseId" = course_id
    ) THEN
        -- If not exists, create a new UserCourse entry with expiry date after 3 days from today
        INSERT INTO "UserCourse" ("userId", "courseId", "expiryAt", "createdAt", "updatedAt")
        VALUES (user_id, course_id, NOW() + INTERVAL '3 days', NOW(), NOW());
    END IF;
END;
$$;

-- Alter function owner
alter function add_reflex_trial_user_course(integer, integer) owner to neetprep_rw;
