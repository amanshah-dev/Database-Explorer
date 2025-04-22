create function "PopulateDailyUserAllEvent"(days integer) returns void
    language plpgsql
as
$$
BEGIN
    PERFORM "PopulateDailyUserEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserTestQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserNcertQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserCorrectQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserPhysicsQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserChemistryQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserPhysicsCorrectQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserChemistryCorrectQEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserTestSubmitEvent"(days, date(current_timestamp));
    PERFORM "PopulateDailyUserVideoViewEvent"(days, date(current_timestamp));
END
$$;

alter function "PopulateDailyUserAllEvent"(integer) owner to learner;
