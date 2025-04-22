-- Create function to add course offer for Offline Test Series Course for Nucleus students
create function "AddCourseOfferForOfflineTestSeriesCourseNucleusStudents"(totalamount integer) returns void
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
    join "UserCourse" on "User"."id" = "UserCourse"."userId" and "UserCourse"."courseId" in (2696, 8, 141) and "UserCourse"."expiryAt" > current_timestamp + interval '1 month'
    where "Payment"."createdAt" > '2023-01-01'
    and "Payment"."status" = 'responseReceivedSuccess'
    and "Payment"."amount" > 2000
    and "Payment"."amount" <= 20000
    and "UserCourse"."createdAt" - "Payment"."createdAt" < interval '3 days' and "UserCourse"."createdAt" > "Payment"."createdAt"
    and "Payment"."createdAt" < current_timestamp - interval '1 week'
    and "User"."email" is not null and not exists (select * from "UserCourse" uc where uc."userId" = "User"."id" and "expiryAt" > now() + interval '1 month' and "courseId" in (3323)) loop
    insert into
    "CourseOffer"("email", "phone", "title", "description", "courseId", "fee", "discountedFee", "expiryAt", "offerStartedAt", "offerExpiryAt", "hidden")
    values(coalesce("t_row"."email", ''), coalesce("t_row"."phone", ''), 'Classroom Test Series offer Target Batch Students', '<p class="course-offer-discount">Enroll in Offline Classroom test series. Last Chance to avail the exclusive discount on your account. Enrollment price increasing soon. You will be allotted the center of your choice from <a href="https://www.neetprep.com/exam-info/Offline-Test-Series-Centers" target="_blank">the list of available centers</a> after payment.</p>', 3323, 9999, totalAmount - 2897, '2024-05-10', current_date, current_timestamp + interval '3 days', true) ON CONFLICT ON CONSTRAINT single_applicable_course_offer do nothing;
end loop;
END
$$;

-- Alter function owner
alter function "AddCourseOfferForOfflineTestSeriesCourseNucleusStudents"(integer) owner to learner;
