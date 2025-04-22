-- Create function to add Prodigy Full Test Course access
create function "AddProdigyFullTestCourseAccess"() returns void
    language plpgsql
as
$$
BEGIN
  insert into "UserCourse" ("userId", "courseId", "expiryAt")
    select t."userId", 3686, '2024-05-05' from
        (select "userId", 3686 from "UserCourse" where "courseId" in (2993, 2960, 3224, 3356) and "expiryAt" > current_timestamp and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '4 week' and exists (select * from "UserCourse" uc where uc."courseId" = 3161 and uc."userId" = "UserCourse"."userId")
        except
        select "userId", 3686 from "UserCourse" where "courseId" in (3686) and "expiryAt" > current_timestamp and "startedAt" > current_timestamp - interval '4 week') t, "UserCourse" uc
    where uc."userId" = t."userId" and "courseId" in (3161) and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '1 year' on conflict ("userId", "courseId", "expiryAt") do nothing;
END
$$;

-- Alter function owner
alter function "AddProdigyFullTestCourseAccess"() owner to learner;
