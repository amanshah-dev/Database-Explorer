-- Create function to add AR course access for old Abhyas students
create function "AddARCourseAccessForOldAbhyasStudents"() returns void
    language plpgsql
as
$$
BEGIN
  insert into "UserCourse" ("userId", "courseId", "expiryAt")
    select t."userId", 3622, '2025-05-10' from
        (select "userId", 3622 from "UserCourse" where "courseId" in (3125, 3653, 4148) and "expiryAt" > current_timestamp and "expiryAt" > "startedAt" + interval '10 days' and "startedAt" > '2024-05-05'
        -- and exists (select * from "UserCourse" uc where uc."courseId" in (3125, 3653) and "expiryAt" > "startedAt" + interval '10 days' and "expiryAt" < '2024-05-07' and uc."userId" = "UserCourse"."userId")
        except
        select "userId", 3622 from "UserCourse" where "courseId" in (3622) and "expiryAt" > current_timestamp and "expiryAt" > "startedAt" + interval '10 days') t, "UserCourse" uc
    where uc."userId" = t."userId" and "courseId" in (3125, 3653, 4148) and "expiryAt" > "startedAt" + interval '10 days' and "startedAt" > '2024-05-05' on conflict ("userId", "courseId", "expiryAt") do nothing;
END
$$;

-- Alter function owner
alter function "AddARCourseAccessForOldAbhyasStudents"() owner to neetprep_rw;
