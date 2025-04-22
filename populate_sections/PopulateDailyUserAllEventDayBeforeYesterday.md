create function "PopulateDailyUserAllEventDayBeforeYesterday"() returns void
    language plpgsql
as
$$
BEGIN
    PERFORM "PopulateDailyUserEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserTestQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserNcertQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserCorrectQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserPhysicsQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserChemistryQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserPhysicsCorrectQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserChemistryCorrectQEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserTestSubmitEvent"(1, date(current_timestamp) - 1);
    PERFORM "PopulateDailyUserVideoViewEvent"(1, date(current_timestamp) - 1);
END
$$;

alter function "PopulateDailyUserAllEventDayBeforeYesterday"() owner to neetprep_rw;
