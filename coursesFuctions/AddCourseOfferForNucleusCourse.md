-- Create function to add course offer for Nucleus Course
create function "AddCourseOfferForNucleusCourse"(totalamount integer) returns void
    language plpgsql
as
$$
DECLARE
   t_row RECORD;
BEGIN
for t_row in
    select distinct "Payment"."id", "Payment"."userId", "amount", "User"."email", "User"."phone"
    from "Payment"
    join "User" on "User"."id" = "Payment"."userId"
    join "UserCourse" on "User"."id" = "UserCourse"."userId" and "UserCourse"."courseId" = 2729 and "UserCourse"."expiryAt" > current_timestamp + interval '1 month'
    where "Payment"."createdAt" > '2023-01-01'
    and "Payment"."status" = 'responseReceivedSuccess'
    and "Payment"."amount" > 2000
    and "Payment"."amount" <= 4000
    and "UserCourse"."createdAt" - "Payment"."createdAt" < interval '3 days' and "UserCourse"."createdAt" > "Payment"."createdAt"
    and "Payment"."createdAt" < current_timestamp - interval '1 week'
    and "User"."email" is not null and not exists (select * from "UserCourse" uc where uc."userId" = "User"."id" and "expiryAt" > now() + interval '1 month' and "courseId" in (8, 141, 2696)) loop
    insert into
    "CourseOffer"("email", "phone", "title", "description", "courseId", "fee", "discountedFee", "expiryAt", "offerStartedAt", "offerExpiryAt", "hidden")
    values(coalesce("t_row"."email", ''), coalesce("t_row"."phone", ''), 'Nucleus Course Offer for Target Batch Students', '<p class="course-offer-discount">Get access to recorded video lectures of all chapters. The discount is based on your previous purchases and valid only for 3 days</p>', 2696, 12999, totalAmount - "t_row"."amount", '2024-05-10', current_date, current_timestamp + interval '3 days', true) ON CONFLICT ON CONSTRAINT single_applicable_course_offer do nothing;
end loop;
END
$$;

-- Alter function owner
alter function "AddCourseOfferForNucleusCourse"(integer) owner to learner;
