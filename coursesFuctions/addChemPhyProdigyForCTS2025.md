-- Create function to add ChemPhy Prodigy for CTS2025 students
create function "addChemPhyProdigyForCTS2025"() returns void
    language plpgsql
as
$$
BEGIN
  insert into "UserCourse" ("userId", "courseId", "expiryAt")
    select t."userId", 3161, "uc"."expiryAt" from
        (select "userId", 3161 from "UserCourse" where "courseId" in (3323, 4215, 4214, 4478, 4676, 4673) and "expiryAt" > current_timestamp and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '4 week'
        except
        select "userId", 3161 from "UserCourse" where "courseId" in (3161) and "expiryAt" > current_timestamp and "startedAt" > current_timestamp - interval '4 week') t, "UserCourse" uc
    where uc."userId" = t."userId" and "courseId" in (3323, 4215, 4214, 4478, 4676, 4673) and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '4 week' on conflict ("userId", "courseId", "expiryAt") do nothing;
END
$$;

-- Alter function owner
alter function "addChemPhyProdigyForCTS2025"() owner to neetprep_rw;
