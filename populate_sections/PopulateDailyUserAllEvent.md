create function "PopulateDailyUserAllEvent"() returns void
    language plpgsql
as
$$
BEGIN
    PERFORM "PopulateDailyUserEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserTestQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserNcertQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserCorrectQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserPhysicsQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserChemistryQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserPhysicsCorrectQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserChemistryCorrectQEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserTestSubmitEvent"(1, date(current_timestamp));
    PERFORM "PopulateDailyUserVideoViewEvent"(1, date(current_timestamp));
END
$$;

alter function "PopulateDailyUserAllEvent"() owner to learner;
