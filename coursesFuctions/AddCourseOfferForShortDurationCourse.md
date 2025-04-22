-- Create function to add course offer for short duration courses
create function "AddCourseOfferForShortDurationCourse"() returns void
    language plpgsql
as
$$
DECLARE 
   t_row RECORD;
BEGIN
for t_row in
  select "Payment"."userId", "amount", "User"."email", "User"."phone"
  from "Payment"
  join "User" on "User"."id" = "Payment"."userId"
  join "UserCourse" on "UserCourse"."id" = "Payment"."purchasedItemId"
  where "Payment"."createdAt" > '2020-06-18' 
  and "Payment"."response" is not null 
  and "Payment"."paymentForId" in (287, 255, 38, 43, 44, 45, 53, 55, 57, 58, 59, 60, 62, 63)
  and "Payment"."status" = 'responseReceivedSuccess'
  and "UserCourse"."courseId" != 8 and "UserCourse"."expiryAt" > current_timestamp
  and "User"."email" is null loop
    insert into 
    "CourseOffer"("email", "phone", "title", "description", "courseId", "fee", "discountedFee", "expiryAt", "offerStartedAt", "offerExpiryAt") 
    values(coalesce("t_row"."email", ''), coalesce("t_row"."phone", ''), 'Title', '<p class="course-offer-discount">Discount Based on your previous purchases</p>', 29, 12999, 5999 - "t_row"."amount", '2021-06-30', current_timestamp, '2021-01-31') ON CONFLICT ON CONSTRAINT single_applicable_course_offer do update set "discountedFee" = case when "CourseOffer"."discountedFee" - "t_row"."amount" > 0 then "CourseOffer"."discountedFee" - "t_row"."amount" else 0 end;
end loop;
END
$$;

-- Alter function owner
alter function "AddCourseOfferForShortDurationCourse"() owner to learner;
