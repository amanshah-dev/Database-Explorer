CREATE FUNCTION "CreateCourseAccessForZeroFee"() RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE 
   t_row RECORD;
BEGIN
  FOR t_row IN
    SELECT "CourseOffer".* 
    FROM "CourseOffer" 
    WHERE "CourseOffer"."accepted" = false 
      AND "CourseOffer"."hidden" = false  
      AND ("email" IS NOT NULL) 
      AND ("courseId" = 29)
      AND ("discountedFee" = 0)
      AND ("offerExpiryAt" > current_timestamp) 
      AND ("offerStartedAt" < current_timestamp) 
      AND ("phone" IS NOT NULL) 
      AND ("offerExpiryAt" > current_timestamp) 
      AND ("offerStartedAt" < current_timestamp) 
  LOOP
    WITH user_row AS (
      SELECT * 
      FROM "User" 
      WHERE "email" = t_row."email"
    )
    INSERT INTO public."UserCourse"
    ("startedAt", "expiryAt", "role", "courseId", "userId", "couponId", "trial", "invitationId", "courseOfferId")
    SELECT current_timestamp, t_row."expiryAt", 'courseStudent', t_row."courseId", user_row."id", NULL, true, NULL, t_row."id"
    FROM user_row;
    
    UPDATE "CourseOffer" 
    SET "accepted" = true 
    WHERE "id" = t_row."id";
  END LOOP;
END
$$;

ALTER FUNCTION "CreateCourseAccessForZeroFee"() OWNER TO learner;
