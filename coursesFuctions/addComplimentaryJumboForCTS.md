-- Create function to add complimentary Jumbo for CTS students
create function "addComplimentaryJumboForCTS"() returns void
    language plpgsql
as
$$
BEGIN
  insert into "UserCourse" ("userId", "courseId", "expiryAt")
    select t."userId", 3160, "uc"."expiryAt" from
        (select "userId", 3160 from "UserCourse" where "courseId" in (3323) and "expiryAt" > current_timestamp and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '1 week'
        except
        select "userId", 3160 from "UserCourse" where "courseId" in (3160) and "expiryAt" > current_timestamp and "startedAt" > current_timestamp - interval '1 week') t, "UserCourse" uc
    where uc."userId" = t."userId" and "courseId" in (3323) and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '1 week' on conflict ("userId", "courseId", "expiryAt") do nothing;
END
$$;

-- Alter function owner
alter function "addComplimentaryJumboForCTS"() owner to neetprep_rw;
