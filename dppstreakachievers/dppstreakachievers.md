CREATE FUNCTION dppstreakachievers(userids BIGINT[], questions INTEGER, dpptype dpp_type_enum) RETURNS BIGINT[]
    LANGUAGE plpgsql
AS
$$
DECLARE
    streakAchievers BIGINT[] := ARRAY []::BIGINT[]; -- Array to store the result
BEGIN
    SELECT ARRAY_AGG("userId")
    INTO streakAchievers
    FROM (SELECT ud."userId"
          FROM "UserDpp" ud
          WHERE "userId" = ANY (userIds)
            AND ud."dppType" = dppType
          GROUP BY "userId"
          HAVING COUNT(*) > questions) subquery;
    RETURN streakAchievers;
END;
$$;

ALTER FUNCTION dppstreakachievers(bigint[], integer, dpp_type_enum) OWNER TO neetprep_rw;
