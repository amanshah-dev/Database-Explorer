-- Create function to add course offer for merit reward
create function "AddCourseOfferForMeritReward"() returns void
    language plpgsql
as
$$
DECLARE 
   t_row RECORD;
BEGIN
for t_row in
   select 
     "User"."email", "User"."phone"
   from 
    "UserCourse"
   join "User" on "User"."id" = "UserCourse"."userId"
   join "Course" on "Course"."id" = "UserCourse"."courseId"
    and "Course"."fee" > 0
    and "UserCourse"."expiryAt"::date > '2022-05-01' 
    and "UserCourse"."expiryAt"::date < '2022-11-30' 
  loop
    insert into 
    "CourseOffer"(
      "email", 
      "phone", 
      "title", 
      "description", 
      "courseId", 
      "fee", 
      "discountedFee", 
      "expiryAt", 
      "offerStartedAt", 
      "offerExpiryAt"
    ) values(
      coalesce("t_row"."email", ''), 
      coalesce("t_row"."phone", ''), 
      'Merit Reward Offer',
      '<p class="course-offer-discount">You have been given access to <a target="_blank" href="/neet-course/1541">AIIMS Level Test Series</a>. So, you should not pay if you do not want to be considered for <a target="_blank" href="/exam-info/merit-reward">MERIT Reward.</a></p>', 
      1508,
      1499, 
      899, 
      '2022-05-31',
      current_timestamp, 
      '2022-05-31'
    ) ON CONFLICT ON CONSTRAINT single_applicable_course_offer do update set "discountedFee" = 899;
 end loop;
END
$$;

-- Alter function owner
alter function "AddCourseOfferForMeritReward"() owner to learner;
