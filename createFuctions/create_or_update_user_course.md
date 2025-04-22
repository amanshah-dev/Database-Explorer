CREATE FUNCTION create_or_update_user_course(in_user_id integer, in_course_id integer, in_expiry_at timestamp with time zone) RETURNS void
    LANGUAGE plpgsql
AS
$$
DECLARE
    active_user_course RECORD;
BEGIN
  INSERT INTO "UserCourse" ("userId", "courseId", "startedAt", "expiryAt")
  VALUES (in_user_id, in_course_id, NOW(), in_expiry_at)
  ON CONFLICT ("userId", "courseId", "expiryAt") DO NOTHING;
END;
$$;

ALTER FUNCTION create_or_update_user_course(integer, integer, timestamp with time zone) OWNER TO neetprep_rw;
