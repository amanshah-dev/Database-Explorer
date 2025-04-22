-- Create function to add ChemPhy Prodigy for Target2024 students
create function "addChemPhyProdigyForTarget2024"() returns void
    language plpgsql
as
$$
BEGIN
  insert into "UserCourse" ("userId", "courseId", "expiryAt") 
    select t."userId", 3161, "uc"."expiryAt" from 
        (select "userId", 3161 from "UserCourse" where "courseId" in (2729, 2532, 2762, 8, 18, 19, 20, 141, 271, 272, 273, 1409, 1410, 1411, 2696, 2862, 2928) and "expiryAt" > current_timestamp and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '4 week'
        except 
        select "userId", 3161 from "UserCourse" where "courseId" in (3161) and "expiryAt" > current_timestamp and "startedAt" > current_timestamp - interval '4 week') t, "UserCourse" uc 
    where uc."userId" = t."userId" and "courseId" in (2729, 2532, 2762, 8, 18, 19, 20, 141, 271, 272, 273, 1409, 1410, 1411, 2696, 2862, 2928) and "expiryAt" > current_timestamp + interval '10 days' and "startedAt" > current_timestamp - interval '4 week' on conflict ("userId", "courseId", "expiryAt") do nothing;
END
$$;

-- Alter function owner
alter function "addChemPhyProdigyForTarget2024"() owner to learner;
